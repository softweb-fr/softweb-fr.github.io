---
title: "Nodejs Module Customization Hooks"
date: 2024-01-04
---

Dans ce tutoriel, nous allons explorer comment importer des fichiers non natifs dans Node.js en utilisant les
["Module Customization Hooks"](https://nodejs.org/docs/latest-v20.x/api/module.html#customization-hooks).

Je vais prendre comme exemple l'importation de fichiers `.graphql` dans un projet Node.js compilé avec esbuild.

## Mise en place des fichiers initiaux

### `assets/schema.graphql`

Exemple de schéma GraphQL :

```graphql
"Description for the type"
type MyObjectType {
  """
  Description for field
  Supports **multi-line** description for your [API](http://example.com)!
  """
  myField: String!

  otherField(
    "Description for argument"
    arg: Int
  )
}
```

### `src/index.js`

Tentons d'importer directement le fichier `.graphql`:

```javascript
import graphql from '../assets/schema.graphql'

console.log({graphql})
```

À ce stade, aucune configuration spéciale n'est mise en place pour gérer l'importation des fichiers `.graphql`.

### `./esbuild.config.js`

Voici la configuration de base pour esbuild :

```javascript
import * as esbuild from 'esbuild'

await esbuild.build({
  entryPoints: ['src/index.js'],
  bundle: true,
  platform: 'node',
  target: ['node20'],
  outfile: 'dist/index.js',
})
```

## Les erreurs que nous obtenons

En essayant d'exécuter le fichier `src/index.js`, vous rencontrez l'erreur suivante :

```console
$ node src/index.js
TypeError [ERR_UNKNOWN_FILE_EXTENSION]: Unknown file extension ".graphql" for assets/schema.graphql
```

En exécutant `./esbuild.config.js`, vous obtenez :

```console
$ node esbuild.config.js
✘ [ERROR] No loader is configured for ".graphql" files: assets/schema.graphql

    src/index.js:1:20:
      1 │ import graphql from '../assets/schema.graphql'
        ╵                     ~~~~~~~~~~~~~~~~~~~~~~~~~~
```

Ces erreurs indiquent que Node.js et esbuild ne savent pas comment traiter les fichiers avec l'extension `.graphql`.

## Solution via Custom Loader

Pour résoudre ces problèmes, nous allons configurer Node.js et esbuild pour comprendre et charger les fichiers
`.graphql` correctement.

Installez [`node-esm-loader`](https://www.npmjs.com/package/node-esm-loader) et
[`js-string-escape`](https://www.npmjs.com/package/js-string-escape) pour aider à charger et à transformer le contenu
`.graphql`.

Créez un fichier `./.loaderrc.js` pour définir comment traiter les fichiers `.graphql` :

```javascript
import jsStringEscape from 'js-string-escape'

export const test = /\.graphql$/
export const loader = content => `export default "${jsStringEscape(content)}"`
export default {test, loader}
```

Modifiez le fichier `./esbuild.config.js` pour intégrer le loader :
    
```javascript
import * as esbuild from 'esbuild'
import {readFile} from "node:fs/promises"
import {resolve} from "node:path"
import {test, loader} from './.loaderrc.js'


await esbuild.build({
  entryPoints: ['src/index.js'],
  bundle: true,
  platform: 'node',
  target: ['node20'],
  outfile: 'dist/index.js',
  plugins: [{
    name: "graphql",
    setup(build) {
      build.onResolve({filter: test}, ({path, resolveDir}) => ({
        path: resolve(resolveDir, path)
      }))
      build.onLoad({filter: test}, async ({path}) => ({
        contents: loader(await readFile(path, "utf8")),
      }))
    }
  }]
})
```

Maintenant nous pouvons exécuter `./esbuild.config.js` et `src/index.js` sans erreur :

```console
$ node --import=node-esm-loader/register src/index.js
{
  graphql: '"Description for the type"\n' +
    'type MyObjectType {\n' +
    '  """\n' +
    '  Description for field\n' +
    '  Supports **multi-line** description for your [API](http://example.com)!\n' +
    '  """\n' +
    '  myField: String!\n' +
    '\n' +
    '  otherField(\n' +
    '    "Description for argument"\n' +
    '    arg: Int\n' +
    '  )\n' +
    '}'
}
```

```console
$ node esbuild.config.js && node dist/index.js
{
  graphql: '"Description for the type"\n' +
    'type MyObjectType {\n' +
    '  """\n' +
    '  Description for field\n' +
    '  Supports **multi-line** description for your [API](http://example.com)!\n' +
    '  """\n' +
    '  myField: String!\n' +
    '\n' +
    '  otherField(\n' +
    '    "Description for argument"\n' +
    '    arg: Int\n' +
    '  )\n' +
    '}'
}
```

Le fichier `dist/index.js` qui est généré par esbuild contient le code suivant :

```javascript
// assets/schema.graphql
var schema_default = '"Description for the type"\ntype MyObjectType {\n  """\n  Description for field\n  Supports **multi-line** description for your [API](http://example.com)!\n  """\n  myField: String!\n\n  otherField(\n    "Description for argument"\n    arg: Int\n  )\n}';

// src/index.js
console.log({ graphql: schema_default });
```

Nous n'avons plus de reférence vers l'asset `assets/schema.graphql` mais le contenu du fichier est directement intégré
dans le fichier `dist/index.js`. Le fichier peut donc est déployé sans avoir à se soucier de la gestion des assets.

## Conclusion

En suivant ces étapes, vous avez réussi à configurer votre projet Node.js pour utiliser des
["Module Customization Hooks"](https://nodejs.org/docs/latest-v20.x/api/module.html#customization-hooks) afin d'importer
et de traiter les fichiers `.graphql`. Cette approche peut être étendue à d'autres types de fichiers non natifs et
permet de simplifier la gestion des assets dans vos projets Node.js.

Vous pouvez retrouver le code source de cet article sur [GitHub](https://github.com/ferjul17/custom-loader-example).