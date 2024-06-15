+++
title = "Creating a web-app with Svelte and Rollup as a game developer"
date = 2019-08-18
path = "2019/08/creating-a-web-app-with-svelte-and-rollup-as-a-game-developer"
+++

I'm not a web developer. I'm a game developer who primarily uses C++. But I needed to set up a nice easy-to-work-with web-app for a project, so I was forced to kinda learn how to do that.

As I'm not well-versed in the world of modern web technologies, I figured I'd write a blog post of my experience of setting up a simple Svelte site, for people who might be in the same boat as me.

<!-- more -->

**NOTE**: I am no longer using Svelte nor Rollup frequently. There are now much better tools that work much easier out-of-the-box, like [Vite](https://vitejs.dev/guide/).

---

I got started by going through the [Svelte tutorial](https://svelte.dev/tutorial/basics). I learned how components work and how to do the basic templating stuff. Great! But then I hit the most demotivating part of the tutorial; setting up a build tool.

The tutorial helpfully links to a “step-by-step guide” to setting this up specifically aimed for “new developers”, however at the time of writing, the link points to an empty “Coming Soon” page. Very useful. Okay, guess I have to figure it out myself.

In this post I will only go into setting up the build tool for Svelte, if you need to learn more about how Svelte itself works, the tutorial is an excellent source of information.

Unfortunately, the good ol' days of web development where you only needed to reference a single .js or a .css file are over. To use a lot of the modern web technology now, you will need to compile your code with some kind of build tool.

If you're used to C++ build tools such as CMake/Premake/GENie/etc, then [rollup](https://rollupjs.org/) should be somewhat familiar. You create a configuration file, and run a command line tool to build your project.

First of all, the majority of the build tools you're going to find in the web tech space is going to be a [Node.js](https://nodejs.org/en/) package. This means that all these build tools run on Javascript themselves. So we install Node.js and continue. Node.js comes with a package manager called “npm”, which we will use to download and install the tools and libraries necessary.

Start by creating a new folder for your project. Then, on the command line, run the following:

```bash
npm install -g rollup
```

This will install the rollup build tool for us, which we will use to build the code to a single .js file. By the way, while C++ build tools produce “binaries”, we call these build products “bundles”, as it is bundling all of our source code together into 1 big javascript source file that we include in our html file.

Note that we passed the `-g` flag to npm while installing rollup. This means that it will install it globally, rather than locally in the project. This is an important detail, as we don't want the entire rollup binary inside of our project folder.

What we do want in our project folder however, is the libraries we'll be using. In our case, Svelte. We will also need to install a “plugin” for rollup that will compile the .svelte files for us. So we run npm again:

```bash
npm install --save-dev svelte rollup-plugin-svelte rollup-plugin-node-resolve
```

Note that we are installing these packges not with `-g`, but with `--save-dev`. This means we will install them locally (by omitting the global flag) and saving the dependencies in the `devDependencies` list in `package.json`. Normal dependencies are dependencies that should be available in production, and development dependencies are ones that we only need during development. We don't need Svelte as a production dependency, because it'll automatically be included in our bundle when the project compiles.

Also note that we installed 2 rollup plugins: `rollup-plugin-svelte`, and `rollup-plugin-node-resolve`. The first one provides the Svelte compiler, and the second one is a helper plugin that tells rollup where Svelte is located so that it can be properly included in the bundle.

Before we begin writing our rollup configuration file, we create some folders that we'll be using for the source. Start by creating a `public` folder in your project folder. This will be the output directory, and where we will write our entry-point `index.html`. Next, create a `src` folder in your project folder, and inside of that, create a `components` folder. We will be putting Svelte components in there.

We create a simple `src/main.js` file, which will load a component into our HTML's body:

```js
import Main from "./components/Main.svelte";
var app = new Main({
	target: document.body
});
```

And a `src/components/Main.svelte` file:

```html
<h1>Hello, world!</h1>
```

Okay, that's all the code done. Now, off to the rollup configuration. In your project's folder, create a file called `rollup.config.js`. This file will be executed by rollup when invoked from the command like with `rollup -c`. Let's start by importing our rollup plugins:

```js
import svelte from "rollup-plugin-svelte";
import resolve from "rollup-plugin-node-resolve";
```

These lines import the `svelte` and `resolve` functions representing the plugin logic. We will need these later in the script.

```js
export default
{
```

Here, we are starting the configuration object for rollup to use. The syntax used here is a little weird, but it technically allows you to have multiple configurations. The `export default` line above can be replaced with `module.exports =` and it will work the same.

```js
	input: "src/main.js",
```

This specifies the main entry point of your application. Rollup will begin analyzing at this file, and from there figure out which files are imported and required for the application.

```js
	output: {
		file: "public/bundle.js",
		format: "iife",
		compact: true
	},
```

Next up is our output configuration. We specify an output filename, `public/bundle.js`, as the file to compile our bundle to. The format is not super interesting, but basically `iife` is a format that encapsulates the entire bundle in a self-executing function. This means that any local variables etc. will not leak out to the global scope. Nice! The `compact` option is optional, but it makes the output bundle file a little bit smaller.

Now, how do we tell rollup how .svelte files are compiled? This is where the plugins come in.

```js
	plugins: [
		svelte({
			include: "src/components/**/*.svelte",
			css: function(css) {
				css.write("public/style.css", false);
			},
		}),

		resolve({
			browser: true
		})
	]
```

Note that we are loading 2 plugins here, `svelte` and `resolve` that each have their own configuration as well.

Svelte's configuration specifies where our components are located (the .svelte files), and a callback for the CSS. I'm not entirely sure why this is a callback function, but that's how it works. We let Svelte write out its CSS file here. (The `false` as the second parameter means that we don't want it to write a source map file.)

The other plugin, `resolve`, simply helps rollup find the dependencies for Svelte inside of the `node_modules` folder. Without this plugin, it won't be able to include it in the bundle file and it will produce build errors.

The configuration file should now look something like this:

```js
import svelte from "rollup-plugin-svelte";
import resolve from "rollup-plugin-node-resolve";

export default
{
	input: "src/main.js",
	output: {
		file: "public/bundle.js",
		format: "iife",
		compact: true
	},
	plugins: [
		svelte({
			include: "src/components/**/*.svelte",
			css: function(css) {
				css.write("public/style.css", false);
			},
		}),

		resolve({
			browser: true
		})
	]
};
```

Now, we can run a rollup build, and everything should be ok!

```
$ rollup -c
src/main.js → public/bundle.js...
created public/bundle.js in 138ms
```

Hooray! You can verify that `bundle.js` contains actual code now. Now we just have to write our `public/index.html` file:

```html
<!DOCTYPE html>
<html>
	<head>
		<title>Svelte Test</title>
		<meta charset="utf-8">
		<link rel="stylesheet" type="text/css" href="style.css">
	</head>
	<body>
		<script src="bundle.js"></script>
	</body>
</html>
```

Open the HTML file in any browser and.. we should have a hello world on our screen! Woo, congratulations!

An important thing to note is that your script tag should be inside of the body tag, otherwise it won't load!

Tip: To get the traditional save-in-editor, alt-tab to browser, F5 ritual, you can run `rollup -c -w` on the command line to watch for file changes automatically.
