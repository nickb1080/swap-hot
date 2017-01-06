# swap-hot

## Rationale
Large Node.js applications can take a (relatively) long time to start, due largely to `require`ing tens of thousands of files. The vast majority of these are from `node_modules` and won't change during a normal development session. It's annoying, and wasteful, to re-require all of these modules every time you want to see a change reflected in an application. This can also be bothersome when running tests -- often you change a single file and need to only run the associated test file, but starting the test suite incurs a several-second delay as the application starts and requires it's dependencies.

## Prior Art
[`hotswap`](https://github.com/rlidwka/node-hotswap) module is sort of clunky -- it has a weird `module.change_code` property you need to use, which seems dangerously easy to commit to source control. Further, it only works for exported properties -- if you export e.g. a class constructor, updates won't work.

## Implementation
There are two primary pieces, the `Watcher` which observes the file system for changes, and the `Swapper` which does the actual hotswapping. In the vast majority of cases these can both run alongside the application code in the same process. However, under some systems -- Vagrant in particular -- file watching doesn't work very well. In this case, the `Watcher` will run on the host system and the `Swapper` will run alongside the application inside the VM. For simplicity, in either case the two pieces communicate over the network. The hotswapping logic is based on [ES6 Proxies](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy) as well as a library I wrote that [intercepts `require()` calls](https://github.com/nickb1080/intercept-require). Basically, whenever a module in the application is required, it's proxied through a [`Swappable`](https://github.com/nickb1080/swap-hot/blob/master/lib/Swappable.js), which can update its contents from source code transparently at any time.

## Usage

Simple case -- hotswap any module when it changes

```js
const swap = require("swap-hot");
const {close} = swap();
// call `close()` to stop hot swapping
```

Complex case -- running a test file with [Mocha]() whenever the associated source file changes

```js
const swap = require("swap-hot");
const {swapper} = swap();

swapper.on("update", function (filename) {
  const testFile = path.join(this._root, "..", "test", filename + ".js");
  if (testFile == null) {
    console.log("no test file found for", filename);
    return;
  }

  const ext = path.extname(testFile);
  const suffix = crypto.randomBytes(4).toString("hex");

  const testContents = fs.readFileSync(testFile, "utf8");
  // trying to re-run a file with mocha that has already been run before in this process
  // seems to "succeed" always with "0 tests run", so just make a temp file to circumvent
  // this unwanted behavior
  const tmpFile = path.join(path.dirname(testFile), `__hotswap-test-${suffix}${ext}`);
  fs.writeFileSync(tmpFile, testContents);

  const m = new Mocha();
  m.addFile(tmpFile);
  m.run(function (err) {
    if (err) {
      console.log("test runner errored!", err);
      throw err;
    }
    fs.unlinkSync(tmpFile);
  });
})
```
