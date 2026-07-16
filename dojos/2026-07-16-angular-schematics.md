# Angular Dojo: Angular Schematics
**Datum:** 2026-07-16
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du verstehst, wie Angular Schematics funktionieren, und kannst ein eigenes Schematic erstellen, das automatisch eine vorkonfigurierte Feature-Komponente mit Service und Route generiert.

## Hintergrund & Theorie
Angular Schematics sind Code-Generatoren, die über die Angular CLI ausgeführt werden (`ng generate`). Sie basieren auf dem `@angular-devkit/schematics`-Paket und arbeiten mit einem **virtuellen Dateisystem (Tree)** – Änderungen werden erst am Ende commit­tet, was atomares Rollback ermöglicht.

Ein Schematic ist eine Funktion, die eine `Rule` zurückgibt – eine Funktion `(tree: Tree, context: SchematicContext) => Tree | Observable<Tree>`. Regeln lassen sich mit `chain()` komponieren:

```typescript
export function mySchematic(options: MySchema): Rule {
  return chain([
    addFiles(options),
    updateNgModule(options),
  ]);
}
```

Wichtige Bausteine:
- **`Tree`** – virtuelles Dateisystem (lesen, schreiben, löschen)
- **`Rule`** – Transformation des Trees
- **`chain(rules)`** – sequentielle Komposition
- **`url('./files')`** + **`apply()`** + **`template()`** – Datei-Templates mit EJS-ähnlicher Syntax (`<%= name %>`, `__name__`)
- **`SchematicsException`** – typsicherer Fehler
- **Schema-JSON** – definiert CLI-Optionen (mit Validierung und Prompts)

Schematics können in einer **Schematic Collection** gebündelt und über `ng add` oder `ng generate my-lib:feature` aufgerufen werden.

## Aufgabe
Erstelle eine kleine Schematic Collection, die über `ng generate feature <name>` ein Feature-Verzeichnis mit drei Dateien anlegt:
- `<name>/<name>.component.ts` (Standalone Component)
- `<name>/<name>.service.ts`
- `<name>/<name>.routes.ts`

### Schritte

1. **Projekt initialisieren**
   ```bash
   npm install -g @angular-devkit/schematics-cli
   schematics blank --name=my-schematics
   cd my-schematics
   npm install
   ```

2. **Schema definieren** – Lege `src/feature/schema.json` an:
   ```json
   {
     "$schema": "http://json-schema.org/schema",
     "$id": "FeatureSchema",
     "title": "Feature Schema",
     "type": "object",
     "properties": {
       "name": {
         "type": "string",
         "description": "Der Name des Features",
         "$default": { "$source": "argv", "index": 0 }
       },
       "path": {
         "type": "string",
         "description": "Zielpfad im Projekt",
         "default": "src/app"
       }
     },
     "required": ["name"]
   }
   ```

3. **Datei-Templates erstellen** – Lege den Ordner `src/feature/files/__name__/` an und füge drei Template-Dateien hinzu:

   `__name__.component.ts.template`:
   ```typescript
   import { Component } from '@angular/core';
   import { CommonModule } from '@angular/common';

   @Component({
     selector: 'app-<%= name %>',
     standalone: true,
     imports: [CommonModule],
     template: `<p><%= name %> works!</p>`,
   })
   export class <%= classify(name) %>Component {}
   ```

   `__name__.service.ts.template`:
   ```typescript
   import { Injectable, inject } from '@angular/core';
   import { HttpClient } from '@angular/common/http';

   @Injectable({ providedIn: 'root' })
   export class <%= classify(name) %>Service {
     private http = inject(HttpClient);
   }
   ```

   `__name__.routes.ts.template`:
   ```typescript
   import { Routes } from '@angular/router';

   export const <%= camelize(name) %>Routes: Routes = [
     {
       path: '',
       loadComponent: () =>
         import('./<%= name %>.component').then(m => m.<%= classify(name) %>Component),
     },
   ];
   ```

4. **Factory implementieren** – Erstelle `src/feature/index.ts`:
   ```typescript
   import {
     Rule, SchematicContext, Tree,
     apply, url, template, move, chain, mergeWith, MergeStrategy
   } from '@angular-devkit/schematics';
   import { strings } from '@angular-devkit/core';
   import { Schema } from './schema';

   export function feature(options: Schema): Rule {
     return (tree: Tree, context: SchematicContext) => {
       const targetPath = `${options.path}/${options.name}`;

       const templateSource = apply(url('./files'), [
         template({
           ...strings,          // classify, camelize, dasherize, …
           ...options,
         }),
         move(targetPath),
       ]);

       return chain([
         mergeWith(templateSource, MergeStrategy.Overwrite),
       ])(tree, context);
     };
   }
   ```

5. **Collection registrieren** – Ergänze `collection.json`:
   ```json
   {
     "$schema": "../node_modules/@angular-devkit/schematics/collection-schema.json",
     "schematics": {
       "feature": {
         "description": "Generates a feature with component, service and routes",
         "factory": "./feature/index#feature",
         "schema": "./feature/schema.json"
       }
     }
   }
   ```

6. **Bauen & testen**:
   ```bash
   npm run build
   schematics .:feature --name=dashboard --path=src/app --dry-run
   ```
   Überprüfe die Ausgabe auf die drei erwarteten Dateien. Entferne `--dry-run` für echtes Schreiben.

## Hints

<details>
<summary>Hint 1 – Dateinamen-Templates</summary>

`__name__` in Dateinamen wird vom `template()`-Operator durch den Wert von `options.name` ersetzt. Die doppelten Unterstriche sind die Standard-Delimiter des Angular-Schematics-Frameworks – du kannst sie in den Options von `template()` über `interpolationStart`/`interpolationEnd` anpassen.

</details>

<details>
<summary>Hint 2 – strings-Hilfsfunktionen</summary>

`@angular-devkit/core` exportiert `strings` mit nützlichen Transformatoren:
- `strings.classify('my-feature')` → `'MyFeature'`
- `strings.camelize('my-feature')` → `'myFeature'`
- `strings.dasherize('MyFeature')` → `'my-feature'`
- `strings.underscore('myFeature')` → `'my_feature'`

Diese werden im `template()`-Spread an den EJS-Kontext übergeben und stehen in `.template`-Dateien direkt als Funktionen zur Verfügung.

</details>

<details>
<summary>Hint 3 – Testen mit Unit Tests</summary>

Der Schematics CLI stellt `SchematicTestRunner` bereit:

```typescript
import { SchematicTestRunner } from '@angular-devkit/schematics/testing';
import * as path from 'path';

const runner = new SchematicTestRunner(
  'my-schematics',
  path.join(__dirname, '../../collection.json')
);

it('should create feature files', async () => {
  const tree = await runner.runSchematic('feature', { name: 'auth' });
  expect(tree.files).toContain('/src/app/auth/auth.component.ts');
  expect(tree.files).toContain('/src/app/auth/auth.service.ts');
  expect(tree.files).toContain('/src/app/auth/auth.routes.ts');
});
```

</details>

## Beispiellösung

```typescript
// src/feature/index.ts – vollständige Implementierung
import {
  Rule, SchematicContext, Tree,
  apply, url, template, move, chain, mergeWith, MergeStrategy,
  SchematicsException,
} from '@angular-devkit/schematics';
import { strings, normalize } from '@angular-devkit/core';
import { Schema } from './schema';

export function feature(options: Schema): Rule {
  return (tree: Tree, context: SchematicContext) => {
    if (!options.name) {
      throw new SchematicsException('Option "name" is required.');
    }

    const featureName = strings.dasherize(options.name);
    const targetPath = normalize(`${options.path}/${featureName}`);

    context.logger.info(`Generating feature "${featureName}" at ${targetPath}`);

    const templateSource = apply(url('./files'), [
      template({
        ...strings,
        ...options,
        name: featureName,
      }),
      move(targetPath),
    ]);

    return chain([
      mergeWith(templateSource, MergeStrategy.Overwrite),
    ])(tree, context);
  };
}
```

```typescript
// src/feature/index_spec.ts – Unit Test
import { SchematicTestRunner } from '@angular-devkit/schematics/testing';
import * as path from 'path';

const collectionPath = path.join(__dirname, '../../collection.json');

describe('feature schematic', () => {
  const runner = new SchematicTestRunner('my-schematics', collectionPath);

  it('erstellt Component, Service und Routes', async () => {
    const tree = await runner.runSchematic('feature', {
      name: 'user-profile',
      path: 'src/app',
    });

    expect(tree.files).toContain('/src/app/user-profile/user-profile.component.ts');
    expect(tree.files).toContain('/src/app/user-profile/user-profile.service.ts');
    expect(tree.files).toContain('/src/app/user-profile/user-profile.routes.ts');

    const component = tree.readContent(
      '/src/app/user-profile/user-profile.component.ts'
    );
    expect(component).toContain('UserProfileComponent');
    expect(component).toContain('standalone: true');
  });

  it('wirft einen Fehler ohne name-Option', async () => {
    await expectAsync(
      runner.runSchematic('feature', { path: 'src/app' })
    ).toBeRejectedWithError(/name.*required/i);
  });
});
```

## Weiterführendes

- **`ng-add` Schematics**: Mit dem Hook `ng-add` kannst du deine Library automatisch konfigurieren, wenn Nutzer `ng add my-lib` ausführen – ideal für Libraries, die `provideApp()` in `app.config.ts` eintragen müssen.
- **Schematic-Komposition**: Schau dir `@schematics/angular` (das offizielle Angular-CLI-Schematic) auf GitHub an – es zeigt, wie komplexe Regeln wie `addRouteDeclarationToNgModule` oder das Aktualisieren von `tsconfig.json` mit dem `Tree`-API umgesetzt werden.
- **Migrations-Schematics**: Mit `ng update` kannst du automatisierte Code-Migrationen ausliefern (`migrations` in `ng-package.json`), die bei Versionsupdates deiner Library ausgeführt werden.
