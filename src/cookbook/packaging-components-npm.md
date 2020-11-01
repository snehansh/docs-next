# Packaging Vue Components for npm

Vue components by nature are meant to be re-used. This is easy when the component is only used within a single application. But how can you write a component once and use it in multiple sites/applications? A common solution is via npm.

By packaging your component to be shared via npm, it can be imported/required into a build process for use in full-fledged web applications:

```js
import MyComponent from 'my-component'

export default {
  components: {
    MyComponent
  }
  // rest of the component
}
```

Or even used via `<script>` tag in the browser directly:

```html
<script src="https://unpkg.com/vue"></script>
<script src="https://unpkg.com/my-component"></script>
...
<my-component></my-component>
...
```

Not only does this help you avoid copy/pasting components around, but it also allows you to give back to the Vue community!

## Can't I Just Share `.vue` Files Directly?

Vue already allows components to be written as a single file. Because a Single File Component (SFC) is already just one file, you might ask:

> "Why can't people use my `.vue` file directly? Isn't that the simplest way to share components?"

It's true, you can share `.vue` files directly, and anyone using a [Vue build](/guide/installation.html#explanation-of-different-builds) containing the Vue compiler can consume it immediately. Also, the SSR build uses string concatenation as an optimization, so the `.vue` file might be preferred in this scenario (see [Packaging Components for npm > SSR Usage](#SSR-Usage) for details). However, this excludes anyone who wishes to use the component directly in a browser via `<script>` tag, anyone who uses a runtime-only build, or build processes which don't understand what to do with `.vue` files.

Properly packaging your SFC for distribution via npm enables your component to be shared in a way which is ready to use everywhere!

## Packaging Components for npm

For the purposes of this section, assume the following file structure:

```
package.json
build/
   rollup.config.js
src/
   wrapper.js
   my-component.vue
dist/
```

### How does npm know which version to serve to a browser/build process?

The `package.json` file used by npm really only requires one version (`main`), but as it turns out, we aren't limited to that. We can address the most common use cases by specifying 2 additional versions (`module` and `unpkg`), and provide access to the `.vue` file itself using the `browser` field. A sample package.json would look like this:

```json
{
  "name": "my-component",
  "version": "1.2.3",
  "main": "dist/my-component.umd.js",
  "module": "dist/my-component.esm.js",
  "unpkg": "dist/my-component.min.js",
  "browser": {
    "./sfc": "src/my-component.vue"
  },
  ...
}
```

When build tools are used, they will pick up on the `module` build. Legacy applications would use the `main` build, and the `unpkg` build can be used directly in browsers. In fact, the [unpkg](https://unpkg.com) CDN automatically uses this when someone enters the URL for your module into their service!

### How do we use this for SSR?

You might have noticed something interesting - browsers aren't going to be using the `browser` version. That's because this field is actually intended to allow authors to provide [hints to bundlers](https://github.com/defunctzombie/package-browser-field-spec#spec), which in turn create their own packages for client-side use. With a little creativity, this field allows us to map an alias to the `.vue` file itself. For example:

```js
import MyComponent from 'my-component/sfc' // Note the '/sfc'
```

Compatible bundlers see the `browser` definition in package.json and translate requests for `my-component/sfc` into `my-component/src/my-component.vue`, resulting in the original `.vue` file being used instead. Now the SSR process can use the string concatenation optimizations to boost performance.

::: warning
Note: When using `.vue` components directly, pay attention to any type of pre-processing required by `script` and `style` tags. These dependencies will be passed on to users. Consider providing 'plain' SFCs to keep things as light as possible.
:::

### How do I make multiple versions of my component?

There is no need to write your module multiple times. It is possible to prepare all 3 versions of your module in one step, in a matter of seconds. The example here uses [Rollup](https://rollupjs.org) due to its minimal configuration, but similar configuration is possible with other build tools - more details on this decision can be found [here](https://medium.com/webpack/webpack-and-rollup-the-same-but-different-a41ad427058c). The package.json `scripts` section can be updated with a single entry for each build target, and a more generic `build` script that runs them all in one pass. The sample package.json file now looks like this:

```json
{
  "name": "my-component",
  "version": "1.2.3",
  "main": "dist/my-component.umd.js",
  "module": "dist/my-component.esm.js",
  "unpkg": "dist/my-component.min.js",
  "browser": {
    "./sfc": "src/my-component.vue"
  },
  "scripts": {
    "build": "npm run build:umd & npm run build:es & npm run build:unpkg",
    "build:umd": "rollup --config build/rollup.config.js --format umd --file dist/my-component.umd.js",
    "build:es": "rollup --config build/rollup.config.js --format es --file dist/my-component.esm.js",
    "build:unpkg": "rollup --config build/rollup.config.js --format iife --file dist/my-component.min.js"
  },
  "devDependencies": {
    "rollup": "^1.17.0",
    "@rollup/plugin-buble": "^0.21.3",
    "@rollup/plugin-commonjs": "^11.1.0",
    "rollup-plugin-vue": "^5.0.1",
    "vue": "^2.6.10",
    "vue-template-compiler": "^2.6.10"
    ...
  },
  ...
}
```

Our changes to `package.json` are complete. Next, we need a small wrapper to export the actual SFC, plus a minimal Rollup configuration, and we're set!

### What does my packaged component look like?

Depending on how your component is being used, it needs to be exposed as either a [CommonJS/UMD](https://medium.freecodecamp.org/javascript-modules-a-beginner-s-guide-783f7d7a5fcc#c33a) JavaScript module, an [ES6 JavaScript](https://medium.freecodecamp.org/javascript-modules-a-beginner-s-guide-783f7d7a5fcc#4f5e) module so it's immediately available to the page. This is accomplished by a `wrapper.js` file which handles the module export and auto-install. That wrapper, in its entirety, looks like this:

```js
// Import vue component
import component from './my-component.vue'

// Declare install function executed by Vue.use()
export function install(Vue) {
  if (install.installed) return
  install.installed = true
  Vue.component('MyComponent', component)
}

// Create module definition for Vue.use()
const plugin = {
  install
}

// Auto-install when vue is found (eg. in browser via <script> tag)
let GlobalVue = null
if (typeof window !== 'undefined') {
  GlobalVue = window.Vue
} else if (typeof global !== 'undefined') {
  GlobalVue = global.Vue
}
if (GlobalVue) {
  GlobalVue.use(plugin)
}

// To allow use as module (npm/webpack/etc.) export component
export default component
```

Notice the first line directly imports your SFC, and the last line exports it unchanged. As indicated by the comments in the rest of the code, the wrapper provides an `install` function for Vue, then attempts to detect Vue and automatically install the component. With 90% of the work done, it's time to sprint to the finish!

### How do I configure the Rollup build?

With the package.json `scripts` section ready and the SFC wrapper in place, all that is left is to ensure Rollup is properly configured. Fortunately, this can be done with a small 16 line rollup.config.js file:

```js
import commonjs from '@rollup/plugin-commonjs' // Convert CommonJS modules to ES6
import vue from 'rollup-plugin-vue' // Handle .vue SFC files
import buble from '@rollup/plugin-buble' // Transpile/polyfill with reasonable browser support
export default {
  input: 'src/wrapper.js', // Path relative to package.json
  output: {
    name: 'MyComponent',
    exports: 'named'
  },
  plugins: [
    commonjs(),
    vue({
      css: true, // Dynamically inject css as a <style> tag
      compileTemplate: true // Explicitly convert template to render function
    }),
    buble() // Transpile to ES5
  ]
}
```

This sample config file contains the minimum settings to package your SFC for npm. There is room for customization, such as extracting CSS to a separate file, using a CSS preprocessor, uglifying the JS output, etc.

Also, it is worth noting the `name` given the component here. This is a PascalCase name that the component will be given, and should correspond with the kebab-case name used elsewhere throughout this recipe.

## When to avoid this pattern

Packaging SFCs in this manner might not be a good idea in certain scenarios. This recipe doesn't go into detail on how the components themselves are written. Some components might provide side effects like directives, or extend other libraries with additional functionality. In those cases, you will need to evaluate whether or not the changes required to this recipe are too extensive.

In addition, pay attention to any dependencies that your SFC might have. For example, if you require a third party library for sorting or communication with an API, Rollup might roll those packages into the final code if not properly configured. To continue using this recipe, you would need to configure Rollup to exclude those files from the output, then update your documentation to inform your users about these dependencies.

## Acknowledgements

This recipe is the result of a lightning talk given by [Mike Dodge](https://twitter.com/webdevdodge) at VueConf.us in March 2018. He has published a utility to npm which will quickly scaffold a sample SFC using this recipe. You can download the utility, [vue-sfc-rollup](https://www.npmjs.com/package/vue-sfc-rollup), from npm. You can also [clone the repo](https://github.com/team-innovation/vue-sfc-rollup) and customize it.