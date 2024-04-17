---
layout: post
title: Experimenting with Svelte & SvelteKit
date: 2023-12-22 10:39 -0500
---

I recently created a fun little site for a podcast I listen to called [DLC](https://www.patreon.com/dlcpod). For this site I took inspiration from some fan sites others have made for the [Connected podcast](https://www.relay.fm/connected/) where they track predictions and results the hosts make throughout the year. In the first episode of the year for DLC, they list out their predictions for what will happen in the games industry that year. Whether we'll hear about certain titles, whether new consoles will be announced, etc. It's a fun show, where they like to be rather bold with their predictions & hopes for the coming year. They also go through what they call "the reckoning" where they review last year's predictions, and determine how good or not good they were with their predictions.

I decided to create a site to track & display this yearly mayhem: [dlcreckoning.com](https://dlcreckoning.com).

My goal was to make this as a fairly static site, since it doesn't need anything complicated for it to work. A single page app would be overkill and would not provide the right user experience. I typically have used React in most of my frontend work over the past 8 years. Prior to that I mostly used jQuery. I have used Angular for some projects, have toyed around with Vue, SolidJS, and Svelte. I decided to use Svelte for this as I liked its ergonomics, and the idea of it pre-compiling JS to be as lightweight and as optimized as possible. [SvelteKit](https://kit.svelte.dev/) is their framework for creating an application with Svelte, and it offers a full solution for doing server side rendering, loading data, routing, etc. It comes setup with Vite for your development environment, so bundling, importing dependencies all work.

## Setting up SvelteKit as a static site

Furthermore, one can configure SvelteKit to build out as a static site using their static adapter. I'd more call this is partially static in that each route is a generated html page, but it still comes with JavaScript code to load the contents of the other pages as you navigate. I found in local development with the static adapter that hot reloads & page refreshes were slow. To get around this I setup local development to use the default adapter, and production to use the static adapter. This way I could get faster reloads in local development, while still building a static output.

```javascript
// svelte.config.js
import autoAdapter from "@sveltejs/adapter-auto";
import { vitePreprocess } from "@sveltejs/kit/vite";
import staticAdapter from "@sveltejs/adapter-static";

const adapter =
  process.env.NODE_ENV === "production" ? staticAdapter() : autoAdapter();

/** @type {import('@sveltejs/kit').Config} */
const config = {
  // Consult https://kit.svelte.dev/docs/integrations#preprocessors
  // for more information about preprocessors
  preprocess: vitePreprocess(),

  kit: {
    // adapter-auto only supports some environments, see https://kit.svelte.dev/docs/adapter-auto for a list.
    // If your environment is not supported or you settled on a specific environment, switch out the adapter.
    // See https://kit.svelte.dev/docs/adapters for more information about adapters.
    adapter: adapter,
  },
};
```

If you use this to build a site, I recommend testing that you can go to each URL directly in your browser, as well as testing the navigation. Another caveat as well is how the pages are created and uploaded into Amazon S3. By default it will create pages as `2024.html`, `home.html`, etc. But your routes in Svelte will expect to resolve to `/2024` or `/home`. This means that navigation either works in local development or in S3, but not both.

Then for building what I did was used a script to run after the build command executes to rename all the html files, effectively to remove the extension. [https://github.com/agmcleod/dlc-reckoning/blob/main/scripts/renameHtmlFiles.js](https://github.com/agmcleod/dlc-reckoning/blob/main/scripts/renameHtmlFiles.js). This script is then run by updating the build command in package.json: `"build": "vite build && node scripts/renameHtmlFiles.js"`.

What I have to do when I upload the files in the AWS UI, is I upload the html files together, and set the content-type to text/html. Otherwise they'll respond as text files, and the browser will try to download them rather than visit them in an open tab. I then upload the built css & js files in a second pass.

## What I like about Svelte

Setting up the routes & components felt very natural. It took me a bit to learn, but I like the hierarchy of layouts, defining their data, routes & nested routes. It's nice that pages will inherit data from parent routes and layouts. For example, loading in predictions from a JSON file in the layout means every page has it. Likewise if i have a route like `/[year]/+page.ts`, that file will load data for other routes under it, such as: `/[year]/about/+page.svelte`.

In each `+page.svelte` file or `components/Component.svelte` file, I like that the template, JS code, and relevant CSS code is all there. It handles CSS class scoping for you. Variables defined in the script tag are all available in the html template portion. Defining properties of the component is pretty simple, you just export a variable in the script tags. The templating language for if statements, and loops is sensible. Binding events is also pretty smooth. It has built in methods for defining dynamic CSS classes. I find it all just comes together rather naturally, and feels great to use. This all works very well with typescript.

Comparing this to React, features like props, event handlers, control flows are also rather easy to use in React. However how you pull CSS in is very open ended. There's multiple ways of doing state management. Then there's the under the hood parts of React where the virtual DOM is more overhead these days than helpful.

## What don't I like about Svelte

Setting up testing is a pain. I ran into similar issues a bit ago when I rebuilt the front end of another project from React into Svelte. It has a simpler frontend than this project, however it also communicates with a backend I built in Rust. Getting MSW setup with vitest was a big pain. A mix of node module types, and versions led to all sorts of errors. It's possible MSW is smoother to setup now. However when setting up this project I ran into trouble with vitest as well.

The instructions on [testing-library](https://testing-library.com/docs/svelte-testing-library/setup/) were mostly complete, but were not quite correct for setting up with a SvelteKit project. I had to do some digging around for vitest templates on Github to find the answer. In the end I setup a [vitest-setup](https://github.com/agmcleod/dlc-reckoning/blob/main/vitest-setup.ts) file which loads the extensions & jest-dom for testing library. I updated the [vite-config](https://github.com/agmcleod/dlc-reckoning/blob/main/vite.config.ts) to reference this setup file. From there I was fairly good to go.

### Reactivity

I find its reactivity a little hard to grok. I have similar complaints with React hooks not being the most clear API. A component will re-render when its properties change. Like if you have a name prop defined as `export let name: string = ''`, any callers to this component passing a new name, the component will update. However if you compute around that name in the JS code, such as `const displayName = name.toUpperCase()`. Using displayName in the template will not re-render. You need to note it as being reactive.

```javascript
let displayName;
$: displayName = name.toUpperCase();
```

They do document this example fairly well. I find it gets more complicated when you start using data defined in `+page.ts`. When you define `$:` for a block of code, which values is it watching for? Usually when a page hasn't re-rendered properly, adding the $: prefix around the necessary code block has fixed it. I just worry about re-rendering when it's not needed as well. I think this is just something I need more experience with, and need to better understand how it works.

## Overall Impressions

I'd like to continue using Svelte. Probably not something I'll bring into my day job, but I do have a lot of respect for it and would like to continue building things with it. SolidJS also has my interest. The reason I didn't choose it for this project is the community is on the smaller side. I ran into similar testing setup issues with it before. Its SvelteKit equivalent is also just in Beta, so i'd like to give it more time in the oven before I play with it.
