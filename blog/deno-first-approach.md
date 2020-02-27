# Deno, first approach

![banner](https://res.cloudinary.com/practicaldev/image/fetch/s--olocXgeK--/c_imagga_scale,f_auto,fl_progressive,h_420,q_auto,w_1000/https://res.cloudinary.com/practicaldev/image/fetch/s--4-_TFW83--/c_imagga_scale%2Cf_auto%2Cfl_progressive%2Ch_420%2Cq_auto%2Cw_1000/https://thepracticaldev.s3.amazonaws.com/i/udips3nsbtgns4tdkvh6.gif)

## Disclaimer
Before starting, it is very important to remember that at the time of writing, Deno is still under development. Therefore, any produced code must be considered unstable due to potential unanticipated changes in the API.
We will therefore use version `0.21.0` as a basis for the next step.

**Finally, it should also be noted that Deno is not intended to replace Node or merge with it.**

## Introduction & Architecture
Deno is a cross-platform runtime, i.e. a runtime environment, based on [`Google's V8` engine](https://v8.dev/), developed with the [`Rust` language](https://www.rust-lang.org/), and built with [`Tokio` library](https://github.com/tokio-rs/tokio) for the event-loop system.

### Node's problems:
Deno was presented by its creator, **Ryan Dahl (@ry)** at the European JSConf in June 2018, just 1 month after the first commits.  
During this presentation, Dahl exposed ten defects in Node's architecture (to which he blames himself). In summary:
- Node.js has evolved with callbacks at the expense of the Promise API that was present in the first versions of V8
- Security of the application context
- GYP (*Generate Your Projects*), the compilation system forcing users to write their bindings (links between Node and V8) in `C++` while V8 no longer uses it itself.
- The dependency manager, NPM, intrinsically linked to the Node `require` system. NPM modules are stored, until now, on a single centralized service and managed by a private company. Finally, the `package.json` file became too focused on the project rather than on the technical code itself (license, description, repository, etc).
- The `node_modules` folder became much too heavy and complex with the years making the module resolution algorithm complicated. And above all, the use of `node_modules` and the `require` mentioned above is a divergence of the standards established by browsers.
- The `require` syntax omitting `.js` extensions in files, which, like the last point, differs from the browser standard. In addition, the module resolution algorithm is forced to browse several folders and files before finding the requested module.
- The entry point named `index.js` became useless after the require became able to support the `package.json` file
- The absence of the `window` object, present in browsers, preventing any isomorphism

Finally, the overall negative point is that Node has, over time, unprioritize the I/O event saturation system to the benefit of the module system.

### Deno's solutions:
Dahl then began to work in Deno with the aim of solving most of Node's problems. To achieve this, the technology is based on a set of rules and paradigms that allow future developments to follow the guideline:

- #### Native TypeScript support
  One of the top most creator's goals, who has a very special interest in the language. Over the years, we have seen Node struggle with maintaining support for new `V8` and `ECMAScript` features without having to break the existing API.  
  It's over with Deno, which gives you the ability to use TypeScript right away without initial configuration of your application. The use is limited to the native configuration of the default compiler. However, a tsconfig.json file can be given to the compiler using the flag `--config=<file>`.

- #### Isomorphism with the Web by supporting `ECMAScript` module syntax and by banishing the `require()` function
  As mentioned above, Node suffers from ineffective dependency resolution; Deno solves the problem by being more explicit, simple, and direct, while complying with standards.  
  `(import * as log from "https://deno.land/std/log/mod.ts";)`

- #### The distant code is retrieved and cached locally
  Like the `node_modules`, the dependencies that are necessary for the proper working project are downloaded and retrieved locally. However, they will not be stored at the project level but rather in Deno's global cache folder. (`~/.deno/src` by default)  
  The same version of a dependency does not need to be re-downloaded regardless of the number of local projects requiring it. Note that this feature is similar to `yarn plug'n'play`.

- #### Specific permissions must be explicitly given by the end user
  Today, security is foundamental in every applications. For that, Deno contains the executable in a sandbox mode where each operation outside the execution context must be authorized. A network access for example must be granted by an explicit "yes" from the user in the CLI or with the `--allow-net` flag.  
  Once again, Deno wants to move closer to Web paradigms. (access to the webcam by website for example)

- #### One delivrable, one executable
  In order to ensure efficient distribution, Deno offers its own bundler (`deno bundle`) creating a single consumable (.js) at the time of delivery and later, a single executable binary (`deno compile`).

- #### Last but not least...
  Deno also aims to always terminate the program in case of unprocessed errors; to have generated JavaScript code compatible with current browsers; to support Promises at the highest level of the application (`top-level await`, supported by V8, on wait on the TypeScript side); to be able to serve over-HTTP at an efficient speed (if not faster than Node).

### What Deno doest not target (at all):
- #### The use of a `package.json`-like manifest
  A dependency management manifest is not required for a code that retrieves its dependencies itself.

- #### The use of an package manager like `npm`
  For the same reasons, `npm` (or equivalent) is not and should not be essential to the development of a Deno application.

- #### Deno / Node isomorphism
  Even if the two technologies use the same language, the designs are not the same and therefore do not allow isomorphic code.


### Le architectural model:
#### Rust
Rust is the language used to encapsulate the V8 engine. It is he who exposes the isolated functionalities through an API that can be used in JavaScript. This link, or **binding**, called `libdeno`, is delivered as is, independently of the rest of Deno's infrastructure, thanks to a Rust module called `deno-core` (a **crate**; https://crates.io/crates/deno) consumed by the command line, the deno-cli.
This crate can be used in your own Rust app if you want to.

The `deno-cli` is the link between the crate core, the TypeScript compiler (hot compilation and cache of the final code), and Tokyo (an event-loop library).


To summarize, here is a diagram of the execution process:
![](https://raw.githubusercontent.com/denoland/deno_website2/master/public/images/schematic_v0.2.png)

#### Tokio
This library written in Rust gives the language the capability of asynchronous programming and event-oriented programming.  
Natively, Rust does not support event loop management, and has, until 2014, used the `libuv` library to perform its I/O operations asynchronously and cross-platform and thus remedy this flaw.  
It should be noted that Node still uses libuv today in its V8 process.

Thus, Tokio became the reference library for all asynchronous event-driven programming in Rust.

From Deno's point of view, Tokio is therefore in charge of parallelizing all the asynchronous I/O performed by the V8 bindings exposed in the `deno-core` isolate (as a reminder, `deno-core` is the standalone Rust crate)

#### V8
Finally, as mentioned several times earlier, the entire architecture is based on the JavaScript interpretation engine. It is regularly updated to follow the needs of the latest versions of TypeScript, among other things. At the time of writing, the version used by Deno is version `7.9.304` from October 14, 2019.

## Ecosystem & First Developments
### Installation :
For several versions now, Deno is available via Scoop for Windows, and via Homebrew for OSX.  
Installation can also be done manually via `cURL` under Shell, especially for Linux which only has this solution for the moment, or via `iwr` under PowerShell for Windows.  
In the same philosophy as the code, Deno is delivered as a single executable.

```bash
## Shell
curl -fsSL https://deno.land/x/install/install.sh | sh

## PowerShell
iwr https://deno.land/x/install/install.ps1 -useb | iex

## Scoop
scoop install deno

## Homebrew
brew install deno
```

Once the installation is complete, launch the command `deno https://deno.land/welcome.ts` to test its proper functioning.

### deno-cli
The command-line interface provides a set of integrated features that allow you to remain immersive in Deno's proprietary development environment. It also and above all allows you to stay in line with standards when you need to offer your library to the community.

Here is a list of the commands currently available:  
- `deno info` allowing to inspect the dependencies of a program from its entry point
- `deno fmt` allowing the code to be formatted with an integrated `Prettier`
- `deno bundle` mentioned earlier, allowing to transpile our application into a single deliverable with dependencies, into an `.js` file (usable by the browser)
- `deno install` allowing to install a Deno app in home folder (`~/.deno/bin` by default) from an URL or from local code 
- `deno types` allowing to generate Deno's TypesScript types for development
- `deno test` allowing to execute the integrated test tool. (Deno integrates its own test library)
- `deno completions` allowing to add autocomplete in the terminal (normally already added during the Deno installation)
- `deno eval` allowing to interpret a file or string containing code executable by Deno
- `deno xeval` (named on the same idea as `xargs`) allowing `deno eval` to run code, but by taking each line coming from `stdin`

### "HelloWorld.ts"
Now let's talk about our first program. At the moment, even if the Deno ecosystem itself offers a range of development tools that can be used on the command line, the VSCode extension catalog (or other editor) remains very poor in features.  
Don't expect a complete developer experience during your first lines of code.

#### Example 1: Grep
This first example is a simple reproduction of grep's behavior and highlights the import of Deno standard libraries, their usage, as well as the manipulation of files and arguments.

In order to group them, dependencies can be declared in a file conventionally called `deps.ts`:
```typescript
import * as path from "https://deno.land/std/fs/path/mod.ts";
export { path };
export { green, red, bold } from "https://deno.land/std/colors/mod.ts";
```
 
Then be imported classically into its `mod.ts` (equivalent to the `index.js` in Node):
```ts
import { path, green, red, bold } from "./deps.ts";
```

An "*http*" import from Deno is the retrieval of a web resource at the time of compilation. Deno currently only supports `http://`, `https://`, and `file://` protocols.

Then, we validate the arguments passed and retrieved directly from the `Deno` global object:
```typescript
if (Deno.args.length != 3) {
  if (Deno.args.length > 3) {
    throw new Error("grep: to much args.");
  } else {
    throw new Error("grep: missing args.");
  }
}
 
const [, text, filePath] = Deno.args;
```

Finally, we parse and iterate the file to bring out the lines containing the pattern you are looking for:
```typescript
try {
  const content = await Deno.readFile(path.resolve(Deno.cwd(), filePath));
 
  let lineNumber = 1;
  for (const line of new TextDecoder().decode(content).split("\n")) {
    if (line.includes(text)) {
      console.log(
        `${green(`(${lineNumber})`)} ${line.replace(text, red(bold(text)))}`
      );
    }
    lineNumber++;
  }
} catch (error) {
  console.error(`grep: error during process.\n${error}`);
}
```

Finally, to launch the application, execute the command `deno grep/mod.ts foo grep/test.txt`  
`foo` being the pattern, and `test.txt` a file containing strings.

#### Exemple 2 : Overkill Gues-A-Number
This second example is a mini game where the goal is to find a number between 0 and 10 from "more" or "less" clues. It highlights the use of a third-party framework, the import of React, and JSX compatibility.

The import of a third party is almost identical to the import of a standard:
```typescript
import Home from "./page.tsx";
import {
  Application,
  Router,
  RouterContext
} from "https://deno.land/x/oak/mod.ts";
import { App, GuessSafeEnum, generate, log } from "./misc.ts";
```

A `.tsx` file being imported, React must be used to be able to run the whole thing. The `page.tsx` file is completed as follows:
```typescript
import React from "https://dev.jspm.io/react";
import ReactDOMServer from "https://dev.jspm.io/react-dom/server";
```

Thanks to the `.tsx` extension and React, we can use JSX to export a component rendered on the server side for example :
```jsx
export default (props: HomeProps = {}) => `<!DOCTYPE html>
  ${ReactDOMServer.renderToString((
  <>
    <Home {...props} />
    <hr />
    <Debug {...props} />
  </>
))}`;
```

You can run this example with the command `deno guessanumber/mod.ts`  
Finally, you can find the complete examples on Github or even run them directly from their *"raw.githubusercontent"* URLs.  
(https://github.com/bios21/deno-intro-programmez)

## Production & Futur
Right now, Deno is not *ready-to-prod*. The main uses being to create command line tools, background task managers, or web servers (like Node), Deno's performance is not at the level Dahl wants it to be.  
However, it is possible to start experimenting with the development of internal tools such as batch scripts for example.  
A real-time benchmark is available on https://deno.land/benchmarks.html

Comit after comit, the benchmarks are updated and compare Deno's performance to that of Node on several levels, such as the number of requests per second (which is the first bottleneck blocking production use), maximum latency, input-output interactions, memory consumption, etc.  
Deno is already better than Node on a few points and keeps improving over time, hoping to finish first in all the tests performed.

### v1.0
In addition to performance, Deno completes the developer experience with a set of essential features and tools for the release of version 1.0 that can be considered ready for production use.

#### Debug
It is not currently possible to debug or inspect an application; something that can be constraining during development. This major feature is mandatory for version 1.0.  
Taking advantage of `V8`, the debug will rely on the `V8InspectorClient` and the *Chrome Devtools* allowing to use the same tools as with any other JavaScript development.

#### API Stabilization
There are and still are some bugs in the API, either in the TypeScript layer or in the `deno-core`. These bugs, although minor, are still blocking the good stability of the whole.  
Being stable does not only mean having a smooth execution, but also having consistent and uniform entry points. Some functions must therefore be reviewed in terms of their name or even their signatures.

#### Clear and explicit documentation
The common problem with any project starting in the background - the Deno documentation is still very light and lacks use cases or explanations on specific topics.  
The official website is currently being redesigned and will soon be completed.

### Futur
Decoupled from the first stable release, additions to the CLI will be made, support for adding native functionality (via modules called *"ops"* crates in Rust) will be provided, as well as, among many other things, ever closer compatibility with the Web world and **ECMA standards** (e.g. by supporting *WebAssembly modules*).

Concerning the CLI, here is a non-exhaustive list of the planned functionalities:
- `deno compile` allowing to compile its entire application into a purely independent binary.
- `deno doc` allowing to generate a JSON structure of the whole code documentation. This JSON will then be standard to Deno and can then be consumed by a visual documentation tool that includes said standard.
- `deno ast` allowing to generate a JSON structure of the *Abstract Syntax Tree (AST)* of the code from a given entry point. The AST can be used by tools like `ESLint` to programmatically analyze the code structure and identify, for example, potential code flaws or memory leaks.
- The `deno lint` which, in combination with `deno fmt`, will make it possible to make the code produced consistent between all the developers and also to improve the quality by ensuring that it is in line with Deno standards. Please note that the linter configuration will not be accessible or modifiable at the moment.

Version 1.0 is very close and the fast pace of development has allowed the team to estimate a release for the end of the year or early January.  
It is important to remember that Deno remains an open source and community project, and that it is up to the community to help by experimenting with the technology, pushing it to its limits, and providing as much data as possible to the developers.
 
## Community & Contribution
Due to its relatively young age, the Deno community is still small. Nevertheless it is growing every day and many developers from Rust or Node are more and more interested in the technology.  
The biggest communities today are Polish (which includes one of the major contributors through **Bartek Iwańczuk ([@biwanczuk](https://twitter.com/biwanczuk))** ), Korean, Chinese or Japanese.  
Meetup groups are therefore gradually being created like **Deno Poland ([@denopoland](https://twitter.com/denopoland))**, or **Denoland Korea ([@denoland_kr](https://twitter.com/denoland_kr))**.  
France is not to be outdone and already has its first group, **Paris Deno ([@ParisDeno](https://twitter.com/ParisDeno))**.  
A newsletter is also available on https://deno.news

From a contribution point of view, there is a lot to be done. Pull requests on official repositories are "simple" to do as a list of missing features and bugs is available at https://github.com/denoland/deno/milestone. In addition, the contribution rules have been written and completed for the occasion.

The TypeScript layer consists of a `core`, a set of standard `deno_std` libraries (https://deno.land/std/README.md), and a set of third-party libraries combined into a single directory to simplify URLs (https://deno.land/x/).  
Contributions made to the standard and the core **must** respect the rules, but this is not the case for third-party libraries.

Contributions can also be made at the development tool level. Indeed, there is still a lot missing to be comfortable and productive such as VSCode extensions or test libraries equivalent to `Jest` or `fast-check` (whether they are ported, "isomorphized", or rewritten).  
Deno needs you, feel free to go ahead and submit your content; many of the libraries offered are ports of existing libraries from Node, Rust, or even Go.

In conclusion, Deno is still in its early stages, but Ryan Dahl is not at his first try.  
Thanks to the new features of version 1.0, the usability of TypeScript, the more and more interesting performances, and last but not least, because of the confident and growing community, Deno will undoubtedly become one of the potential trending technologies to capitalize on for 2020/2021.  
Stay tuned!