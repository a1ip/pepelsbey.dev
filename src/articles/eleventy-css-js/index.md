When I started to build my first website on [Eleventy](https://www.11ty.dev/) around 2019, I had to decide how to deal with the HTML, CSS, and JavaScript post-processing. By that time, I had gotten used to the convenience of the modular approach and automation. So it wasn’t an option to just copy files from _src_ to _dist_. I needed them stitched, modified, and minified.

By then, I got over the preprocessors like Sass, so I needed some light post-processing for vanilla-flavored CSS and JS. I looked through the [Eleventy starter projects](https://www.11ty.dev/docs/starter/) but couldn’t find anything that would make sense. It was clear that early adopters of Eleventy were struggling with processing resources too.

I came up with a solution that worked for me for a few years, but I recently took the next step, finally making CSS and JS first-class citizens in Eleventy for me 😎

## Vanilla all the way

The first solution I came up with was pretty straightforward. Before setting up any post-processing, I built a system that didn’t require any: I just linked _index.css_ and _index.js_ files to my HTML pages that, in turn, were importing other blocks/modules:

```css
/* index.css */
@import 'blocks/page.css';
@import 'blocks/header.css';
@import 'blocks/content.css';
```

In the JS case, you also need to add `type="module"` to your `<script>` to make it work. Oh, and for some reason, browsers need a `./` prefix for relative ESM imports, but otherwise, it looks pretty much the same:

```js
/* index.js */
import './modules/menu.js';
import './modules/video.js';
import './modules/podcast.js';
```

And you know what? It just worked right in the browser. Yes, it wasn’t ideal from the performance and compatibility perspectives, but that was good enough for local development. And it was super quick too. When modern bundlers argue which one’s faster, I often think that the fastest is the one that’s not running at all 😉

So when I was starting a project for local development via `npm start`, it was Eleventy running its server, watching for changes, and copying CSS and JS files when needed. It would never work for Sass or TypeScript, but I always try to pick the simplest tools for simple tasks.

As for the HTML, Eleventy has been taking care of it for me, generating markup from Markdown, Nunjucks, and data files. On top of that, using built-in `addTransform`, I added a minification processing with [html-minifier-terser](https://github.com/terser/html-minifier-terser).

```js
const htmlmin = require('html-minifier-terser');

config.addTransform('html-minify', (content, path) => {
    if (path && path.endsWith('.html')) {
        return htmlmin.minify(
            content, {
                collapseBooleanAttributes: true,
                collapseWhitespace: true,
                decodeEntities: true,
                includeAutoGeneratedTags: false,
                removeComments: true,
            }
        );
    }

    return content;
});
```

⚠️ This is a part of the bigger Eleventy config, [see the docs](https://www.11ty.dev/docs/config/).

And though I liked this approach a lot for local development, I needed some post-processing for CSS and JS to make the code production-ready.

## Second pass

Running `npm run build` would give me a fully-functional website in the _dist_ folder. I only needed to post-process some files before the deployment. As I always liked the [Gulp](https://gulpjs.com/) for its simplicity, it was an obvious choice. I took [PostCSS](https://postcss.org/) with some plugins for CSS and [Rollup](https://rollupjs.org/) with [Babel](https://babeljs.io/) and [Terser](https://terser.org/) plugins for JS.

Here’s an example of a Gulp task for CSS. Don’t focus on the list of plugins just yet. We’ll have a closer look at them a bit later.

```js
const styles = () => {
    return gulp.src('dist/styles/index.css')
        .pipe(postcss([
            require('postcss-import'),
            require('postcss-media-minmax'),
            require('autoprefixer'),
            require('postcss-csso'),
        ]))
        .pipe(gulp.dest('dist'));
};
```

I enjoyed the whole system for a while because I didn’t have to build and support the Eleventy/Gulp coupling. But at the same time, the additional build step bothered me. I’ve been trying different approaches, from complicated npm scripts to the various [Vite](https://vitejs.dev/) plugins for Eleventy, but they all didn’t make me happy.

## Custom Handlers

But then I noticed something interesting in the [Eleventy v1.0.0 changelog](https://github.com/11ty/eleventy/releases/tag/v1.0.0) 😲

> Custom File Extension Handlers: applications and plugins can now add their own template types and tie them to a file extension.

It didn’t exactly say, “now you can post-process your CSS and JS,” but later, I discovered an [example in the documentation](https://www.11ty.dev/docs/languages/custom/#example-add-sass-support-to-eleventy) adding Sass support to Eleventy. That was precisely what I needed! Not the Sass, but built-in support for processing resources.

Unlike the previous approach, Eleventy takes care of all CSS and JS resources this time. Not only for a production build but also during development. The closer your development build to the production one, the sooner you can spot any problems. But it only works if build time and live reload are fast enough not to get in the way during development.

Every library got its quirks: different APIs, sync/async behavior, etc. So it took me some time to figure out how to make custom handlers work with the libraries I used to process CSS and JS. Like in the Gulp case, I chose [PostCSS](https://postcss.org/) for CSS. For JS, I decided to try [esbuild](https://esbuild.github.io/), known for extremely fast build time, since I needed it to work for both production and development.

Let’s dive into each custom handler to see how they work. I copied them from [this website’s Eleventy config](https://github.com/pepelsbey/pepelsbey.dev/blob/main/eleventy.config.js) and simplified it a little.

### CSS

Remember the file structure? We have _src/styles/index.css_ file that’s importing other CSS files relative to its location. We need to combine them, process a bit, and output a single _dist/styles/index.css_ file.

In the first step, I import all the packages I need to process my styles:

- [postcss](https://www.npmjs.com/package/postcss) to process files using plugins
- [postcss-import](https://www.npmjs.com/package/postcss-import) to stitch all imported files together
- [postcss-media-minmax](https://www.npmjs.com/package/postcss-media-minmax) to polyfill modern Media Query syntax
- [autoprefixer](https://www.npmjs.com/package/autoprefixer) to prefix properties based on the [browserslist](https://browserslist.dev/) config
- [postcss-csso](https://www.npmjs.com/package/postcss-csso) to minimize the result

The PostCSS [plugin ecosystem](https://postcss.org/docs/postcss-plugins) is quite extensive: you can build yourself the whole Sass or Stylus or use CSS from the future specs via [postcss-preset-env](https://github.com/csstools/postcss-plugins/tree/main/plugin-packs/postcss-preset-env) pack of plugins, but I prefer to write CSS that’s already supported in browsers and post-process it for better compatibility.

By default, CSS files are not processed by Eleventy. To process them, we need to add CSS to the template formats list using the `addTemplateFormats` method:

```js
config.addTemplateFormats('css');
```

Now Eleventy is ready to process our CSS files and output the result. Let’s configure this processing. Otherwise, it won’t be different from the passthrough copy. Using `addExtension` we specify what files we’re going to process, including output file extension and async function that will be called with each file’s content and path.

```js
config.addExtension('css', {
    outputFileExtension: 'css',
    compile: async (content, path) => {
        // Processing
    }
});
```

But we don’t need to process every CSS file that Eleventy could find in the _src_ folder. We need only the _index.css_ one, the rest CSS files will be imported into this one. That’s exactly what we‘re going to do next: filter out every other file that’s not the _index.css._

```js
if (path !== './src/styles/index.css') {
    return;
}
```

Now we can finally start processing our _index.css._ And with the `path` passed into the outer function, we can ask PostCSS to figure out the relative location of the rest of the files. It won’t just work otherwise. I learned it the hard way 🥲

```js
return async () => {
    let output = await postcss([
        postcssImport,
        postcssMediaMinmax,
        autoprefixer,
        postcssCsso,
    ]).process(content, {
        from: path,
    });

    return output.css;
}
```

Our files will be stitched together, polyfilled, prefixed, and minified, just like we specified in the list of PostCSS plugins. The output will be passed to Eleventy, which will write it to the _dist/styles/index.css._

<details>
    <summary>The final result</summary>

```js
const postcss = require('postcss');
const postcssImport = require('postcss-import');
const postcssMediaMinmax = require('postcss-media-minmax');
const autoprefixer = require('autoprefixer');
const postcssCsso = require('postcss-csso');

config.addTemplateFormats('css');

config.addExtension('css', {
    outputFileExtension: 'css',
    compile: async (content, path) => {
        if (path !== './src/styles/index.css') {
            return;
        }

        return async () => {
            let output = await postcss([
                postcssImport,
                postcssMediaMinmax,
                autoprefixer,
                postcssCsso,
            ]).process(content, {
                from: path,
            });

            return output.css;
        }
    }
});
```

</details>

### JavaScript

The same goes for JS files: we add template format, then process only _src/scripts/index.js_ using _esbuild_ with some simple options. This function will return the contents of all modules as a single file for Eleventy to output into the _dist_ folder. Unfortunately, _browserslist_ is not supported by _esbuild,_ but it seems like the `es2020` target is similar to what I have there.

```js
return async () => {
    let output = await esbuild.build({
        target: 'es2020',
        entryPoints: [path],
        minify: true,
        bundle: true,
        write: false,
    });

    return output.outputFiles[0].text;
}
```

<details>
    <summary>The final result</summary>

```js
const esbuild = require('esbuild');

config.addTemplateFormats('js');

config.addExtension('js', {
    outputFileExtension: 'js',
    compile: async (content, path) => {
        if (path !== './src/scripts/index.js') {
            return;
        }

        return async () => {
            let output = await esbuild.build({
                target: 'es2020',
                entryPoints: [path],
                minify: true,
                bundle: true,
                write: false,
            });

            return output.outputFiles[0].text;
        }
    }
});
```

</details>

* * *

Not sure if I convinced you, but I’m planning to use this approach for all my future projects based on Eleventy. I might even update the existing ones to make them more maintainable. Though I’m pretty sure there are many other use cases or ways to do it. I’m curious to see what you might come up with! 🙃
