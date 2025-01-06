# cli-primer

__CLI-Primer is a speed wrapper for your next Command Line Interface application.__ 

## Core Features
This package provides you with a convenient entry point (the function `wrapAndRun` in `index.js`) that sets up the most common aspects of a CLI application:
- **Argument Parsing**: reading and validating command-line arguments.
- **Help Documentation**: generating and displaying help documentation based on argument definitions.
- **Configuration File Handling**:
  - **Creating**: initializing a configuration file in user's home directory.
  - **Loading**: reading settings stored in the configuration file and merging them with the command-line arguments, if any, or entirely substituting them.
- **File System Helpers**: setting and cleaning up directory structures, including providing simple template-based file content generation.
- **Built-in Monitoring and Debugging**: conveniently receiving information about events happening virtually everywhere in your application.
- **Termination handling**: providing you the option to run custom code when your application is terminated via `Ctrl+^` or equivalent.
- **Basic session support**: blocking concurrent executions that target the same output directory, when that might be a concern.

## Modules

You also get the complete freedom and flexibility of _not using_ the wrapper function at all. Instead, you can call directly as many (or as little) of the various utility functions made available by this package through its various modules, such as:

### argTools.js

- **`getArguments(dictionary, defaults = {}, monitoringFn = null)`**: 
Parses command-line arguments according to a specified dictionary of expected arguments.

- **`getHelp(dictionary, monitoringFn = null)`**: 
  Generates and returns a help string based on the dictionary of expected arguments.

### configTools.js

- **`getConfigData(filePath, profileName, dictionary, monitoringFn = null)`**: 
  Reads a configuration file and returns the settings of a specified profile.

- **`initializeConfig(filePath, template, templateData, monitoringFn = null)`**: 
  Creates and initializes a configuration file from a given template, transparently injecting a `settings` section that supports multiple profiles.

### session.js
- **`canOpenSession (targetFolder)`**:
  Checks for an existing `operation_in_progress.lock` file in the given `targetFolder` and causes an early exit if found.

- **`openSession(targetFolder, monitoringFn = null)`**:
  Places a `operation_in_progress.lock` file in the given `targetFolder`, effectively preventing concurrent executions that target he same target folder.

- **`closeSession(targetFolder, monitoringFn = null)`**:
  Removes the `closeSession(targetFolder, monitoringFn = null)` file from the given `targetFolder`, ensuring further executions targeting that folder are possible.

### utils.js
- **`monitoringFn(info)`**:
  A simple monitoring function that prints to the console and returns a Boolean indicating whether the received data denotes an error. It supports enabling or disabling debug messages.

- **`setDebugMode(value)`**:
  Toggles the display of `debug` messages by the `monitoringFn`.

- **`getUserHomeDirectory()`**
  Returns the absolute local path to the home directory of the user executing your application.

- **`isWindows()`**:
  Returns `true` if your application is running on Windows, and `false` for any Unix-based OS.

- **`generateUniqueId()`**:
  Returns a convenient, non-cryptographically safe unique ID that you can use to reasonably ensure unicity of various entities, such as files your application generates.

- **`generatePathSafeName(name)`**:
  Returns a path-friendly name that, by best effort, reflects the provided `name`.

- **`getAppInfo(monitoringFn)`**:
  Reads the `package.json` file of the current Node.js application and returns an object with `name`, `author`, `version`, and `description`.

- **`getDefaultBanner(appInfo)`**:
  Generates a default application banner based on the provided `appInfo` object, with minimal error handling.

- **`ensureSetup(homeDir, bluePrint, monitoringFn = null)`**: 
  Ensures a specific folder structure exists and populates it with files based on templates.

- **`removeFolderContents(folderPath, patterns = [], monitoringFn = null)`**: 
  Removes content of a specified folder without deleting the folder itself.

- **`populateTemplate(template, data)`**: 
  Populates a template string with data from an object.

- **`mergeData(intrinsic = {}, implicit = {}, explicit = {}, given = {})`**: 
  Performs a shallow merge of up to four given datasets, giving precedence to the later sets.

- **`deepMergeData(setA, setB)`**:
  Performs deep merge of two given datasets selectively (only nested Objects are merged), giving precedence to the later set.

<br/>

> ⚠ **Note**: all the functions in this package's modules have extensive JsDoc documentation of their own. __Read that__, and you will be up and running in no time.

> ⚠ **Note**: using any of the utility function with or within the `wrapAndRun` wrapper function is perfectly possible. They are not mutually exclusive in any way.

## How to Use

### 1. Install the Package

Install `cli-primer` via `npm`, then require it in your Node.js application. CLI-primer is a CommonJS module.
```bash
npm i cli-primer
```
```javascript
const primer = require('cli-primer');
```

### 2. Wrap your application
In your `index.js` file, call the `wrapAndRun` function passing it a `settings` Object, your application's `main` function, and optionally a function to be called when your app is `CTRL+^`-ed. You can also define a custom _monitoring function_ and pass that as the forth argument, but if you leave that out the `monitoringFn` from `utils.js` will be used and will work just fine.


```javascript
const { wrapAndRun } = require("cli-primer");

// We assume we placed our application's main function in its own module:
const { mainFn } = require("./own_modules/main");

const settings = { /* See next section for details. */ };
(async function () {
  const exitValue = await wrapAndRun( settings, mainFn );
  if ([0,1,2].includes (exitValue)) {
    process.exit(exitValue);
  }
})();
```
See the documentation of `wrapAndRun` in `index.js` for more details.

#### 2.1. The `mainFn`
The `wrapAndRun` function will only call your provided `mainFn` if no fatal error occurred while reading and validating input data, setting up target directory, generating any prerequisite files, and so on. Your provided  `mainFn` function will be called with three arguments, and must have this signature:

```javascript
const numExitVal = myMainFn (inputData, utils, monitoringFn);
```
The `inputData` will contain the merged dataset CLI-primer has built out of your application's configuration file and provided command-line arguments, whichever given. See documentation in `configTools.js` and `argTools.js` for details.

The `utils` will contain the merged set of utility functions CLI-primer provides, across all of its modules. This is just for your convenience. You can still use destructuring to selectively require only the utility functions you need from `"cli-primer"`.

The `monitoring` function is what you have provided as the third parameter when calling `wrapAndRun`, or the default `monitoringFn` function defined in the `utils.js` (see its own documentation there). It is recommended that you inject the monitoring function as the last, optional argument of all the important functions you create, e.g.:

```javascript
function myCoreFunction (myArg1, myArg2, monitoringFn = null) {

  // Safely default to a no-op function if no `monitoringFn` is available.
  const $m = monitoringFn || function () {};

  const foo = ++myArg1;
  const bar = { a:myArg1, b:myArg2 }

  // Use the injected monitoring function to print to the console.
  $m({
    type: "debug",
    message: `Our "foo" is: "${foo}".`,
    data: bar // This will print content of `bar`, formatted, to the console.
  });
}
```
> ⚠ **Note**: remember that you can toggle messages of type `debug` via `setDebugMode(true|false)`.

#### 2.2. The `settings` Object
You can turn on or off the various aspects of CLI-primer by using the flags in this Object. It is still here were you provide the list of the arguments your application understands (this list works both for the values provided via the configuration file or the command line), and still here you can provide a set of optional intrinsic defaults, a template for generating the configuration file, a starting structure for your output directory, and so on.

```javascript
// Example of a fictive `settings` Object:
const settings = {
  showDebugMessages: true,
  useOutputDir: true,
  useSessionControl: false,
  useConfig: true,
  useHelp: true,
  argsDictionary: [
    { name: 'Dry Run', payload: '--isDryRun', doc: 'Prevents actual changes. For debug.' },
    { name: 'Source File', payload: /^--(source|src)=(.+)/, doc: 'File to read from.', mandatory: true },
    {
      name: 'Parse Model',
      payload: /^--(parseModel)=(saasFile|raw)/,
      doc: 'Sets the parsing model to use; one of "saasFile" or "raw".'
    }
  ],
  intrinsicDefaults: {
    parseModel: 'raw'
  },
  configTemplate: JSON.stringify({
    appInfo: {
      appName: "{{name}}",
      appAuthor: "{{author}}",
      appVersion: "{{version}}",
      appDescription: "{{description}}",
    },
  }),
  outputDirBlueprint: {
    content: [
      { type: "folder", path: "relative-path/to/my/folder" },
      {
        type: "file",
        path: "relative-path/to/my/folder/my_file.txt",
        template: "Hello world! My name is {{firstName}} {{lastName}}.",
        data: { firstName: "John", lastName: "Doe" },
      },
      { type: "folder", path: "relative-path/to/my-other/folder" },
    ],
  },
};
```
See the documentation of `wrapAndRun` in `index.js` for more details. Also you should see relevant functions documentation in `argTools.js`, `configTools.js`, `session.js` and `utils.js`.

#### 2.3. The `cleanupFn`
This is an optional third argument that you can pass to `wrapAndRun`. If provided, it will receive the same parameters as `mainFn`. The `cleanupFn` will be invoked when your application is terminated via `CTRL+^` or similar. See the documentation of `wrapAndRun` in `index.js` for more details.

#### 2.4. The return value of the `mainFn`
The function `wrapAndRun` returns a numerical value. If any error occurs before `wrapAndRun` gets to your provided `mainFn`, that number will be `2`, to signal an error. If your application makes use of built-in _early exit_ arguments, such as `--help` (which prints generated documentation to the console and exits), then the exit number will be `1`, to signal an expected early exit. Otherwise, if your `mainFn` gets to be called, the number returned will be whatever number your `mainFn` returns.

Recall that in our first code example, the returned value of `wrapAndRun` was intelligently passed to `process.exit`:
```javascript
  //...
  const exitValue = await wrapAndRun(settings, mainFn);
  if ([0,1,2].includes (exitValue)) {
    process.exit(exitValue);
  }
  //..
```
While not required, doing so will benefit you when users integrate your CLI application in system scripts, which thus get a simple way of knowing whether your application completed normally or not.

In the above example, you could return something else, e.g., `3` from your provided `mainFn`, which would __not__ explicitly call `process.exit` when `wrapAndRun` returns. This can be useful if you intend to run any sort of server or background service from your `mainFn`.

#### 2.5. The return value of the `cleanupFn`
If provided, the `cleanupFn` will act as an interceptor for the `SIGINT` or `SIGTERM` signals your application might receive, giving you a chance to do last-minute cleanup before a user-requested early exit. The return value of your `cleanupFn`, if any, will be passed to `process.exit()`. If `cleanupFn` is not defined, CLI-primer will assume `1` as your application's exit value when terminated via these signals.

> ⚠ **Note**: if you decided to use the basic session control CLI-primer provides, you **do not need to close the session in `cleanupFn`**. CLI-primer always closes sessions for you (if applicable) when your application is terminated early by the user.

> ⚠ **Note**: if the code in your `mainFn` causes an exception, CLI-primer will catch and print that to the console, and **also close the session for you** (if applicable).

### Contribution and Development
Feel free to contribute to cli-primer by submitting [issues](https://github.com/ciacob/cli-primer/issues) or pull requests. The goal is to keep this toolkit simple yet powerful for CLI app development.
