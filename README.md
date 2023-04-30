# React Native Monorepo with yarn workspace

this repo is about mono repo with yarn workspace on react native project. thank you [mmazzarolo](https://mmazzarolo.com/) for your blog about setup monorepo react native. almost in readme is from your blog. I write because I need to summarize about my understanding monorepo react native. no intending to copy. 

## Basic folder structure

```
└── <project-root>/
    ├── package.json
    └── packages/
        # React Native JavaScript code shared across the apps
        ├── shared/
        │   ├── src/
        │   └── package.json
        # Android/iOS app configuration files and native code on app1
        ├── app1/
        │   ├── android/
        │   ├── ios/
        │   ├── app.json
        │   ├── babel.config.js
        │   ├── index.js
        │   ├── metro.config.js
        │   └── package.json
        # Android/iOS app configuration files and native code on app2
        └── app2/
           ├── android/
           ├── ios/
           ├── app.json
           ├── index.js
           ├── metro.config.js
           └── package.json
```

## Getting Started For Setting up Project


### 1. setup monorepo


```
# Create the project dir and cd into it.
mkdir my-app && cd my-app

# Initialize git.
git init
npx gitignore node
```
Add  `package.json`  file:

```
// my-app/package.json
{
  "name": "my-app",
  "version": "0.0.1",
  "private": true,
  "workspaces": {
    "packages": ["packages/*"]
  },
  "nohoist": ["**/react", "**/react-dom"],
  "scripts": {
    "reset": "find . -type dir -name node_modules | xargs rm -rf && rm -rf yarn.lock"
  }
}
```
- The workspaces.packages setting tells Yarn that each package (e.g., app1, app2, etc.) will live in <root>/packages/.
- The reset script deletes all the node_modules directories in the project (recursively) and the root yarn.lock file. It may come in handy during the initial phase of the setup if we mistakenly install dependencies that should be nohoisted before adding them to `nohoist`
- This `nohoist` section will tell Yarn that the listed dependencies should be installed in the `node_modules` directory of each package instead of the root project’s one.


### 2. setup shared workspace


Create an empty `packages` directory:
```
mkdir packages
```
Inside `packages` folder lets’ start from the `shared` React Native JavaScript code.
The idea here is to isolate the JavaScript code that runs the app in an app workspace.
We should think about this workspaces as a standard npm library that can work in isolation.
So it will have its own package.json where we’ll explicitly declare its dependencies.

Create folder `shared` directory:

``` 
mkdir packages/shared && cd packages/shared
```

Add `package.json` file:

```
// my-app/packages/shared/package.json
{
  "name": "@my-app/shared",
  "version": "0.0.0",
  "private": true,
  "main": "src",
  "peerDependencies": {
    "react": "*",
    "react-native": "*"
  }
}
```
we’re setting `react` and `react-native` as peerDependencies because we expect each app that depends on our package to provide their versions of these libraries.

Then, let’s create example component in `shared` workspace in `src/components/ExampleComponent.tsx`

```
// src/components/ExampleComponent.tsx
import React, { useEffect, useState } from "react"
import { Button, StyleSheet, Text, View, Image } from "react-native"
import { useAsyncStorage } from "@react-native-async-storage/async-storage"
import LogoSrc from "../images/logo.png"

// An example demonstrating the usage of native modules. 
// (Yes, it can be optimzed, but that's not the point of the example :P)
export function ExampleComponent(): JSX.Element {
    const [value, setValue] = useState("     ");
    const { getItem, setItem } = useAsyncStorage("@counter");

    const readItemFromStorage = async () => {
        const item = await getItem();
        setValue(item || "");
    };

    const writeItemToStorage = async (newValue: string) => {
        await setItem(newValue);
        setValue(newValue);
    };

    useEffect(() => {
        readItemFromStorage();
    }, []);

    return (
        <View style={styles.root}>
             <Image style={styles.logo} source={LogoSrc} />
            <Text
                style={styles.text}
            >{`Use the button below and refresh the app to test the async-storage native module`}</Text>
            <View style={styles.row}>
                <Text style={styles.text}>Current value: </Text>
                <Text style={styles.value}>{value} </Text>
            </View>
            <Button
                onPress={() =>
                    writeItemToStorage(Math.random().toString(36).substr(2, 5))
                }
                title="Update value"
            />
        </View>
    );
}

const styles = StyleSheet.create({
    root: {
        flex:1,
        justifyContent: 'center',
        alignItems: "center",
        marginTop: 28,
    },
    logo: {
        width: 120,
        height: 120,
        marginBottom: 20,
    },
    row: {
        flexDirection: "row",
        alignItems: "center",
        justifyContent: "center",
        textAlign: "center",
    },
    text: {
        maxWidth: 420,
        marginBottom: 12,
        fontSize: 22,
        fontWeight: "400",
        textAlign: "center",
    },
    value: {
        marginBottom: 12,
        fontSize: 22,
        fontWeight: "600",
        textAlign: "center"
    },
});
```

export ExampleComponent component into `index.ts` on `src`

```
// src/index.ts
export { ExampleComponent } from './components/ExampleComponent'
```
another thing in the `ExampleComponent` it have library that need to config in nohoist and setting in peerDependencies is `@react-native-async-storage/async-storage` because this library run in metro bundler can not able to follow symlink.
so we need to setting `@react-native-async-storage/async-storage` at `my-app/package.json` and `my-app/packages/shared/package.json` like this.

```
// my-app/package.json
{
  "name": "my-app",
  "version": "0.0.1",
  "private": true,
  "workspaces": {
    "packages": ["packages/*"]
  },
  "nohoist": [
    "**/react", 
    "**/react-dom",
    "**/@react-native-async-storage/async-storage", // <- Add this line 
    ],
  "scripts": {
    "reset": "find . -type dir -name node_modules | xargs rm -rf && rm -rf yarn.lock"
  }
}
```

```
// my-app/packages/shared/package.json
{
  "name": "@my-app/shared",
  "version": "0.0.0",
  "private": true,
  "main": "src",
  "peerDependencies": {
    "@react-native-async-storage/async-storage": "*", // <- Add this line 
    "react": "*",
    "react-native": "*"
  }
}
```

So Thanks to Yarn Workspaces, we can now use `@my-app/shared` in any other worskpace by:

- Marking `@my-app/shared` as a dependency
- Importing ExampleComponent: `import { ExampleComponent } from "@my-app/shared"`


### 3. setup app1 workspace

Now that the shared React Native code is ready let’s create packages/app1. This workspace will store the Android & iOS native code and import & run `packages/shared.

Using React Native CLI, bootstrap a new React Native app within the packages directory.

```
cd packages && npx react-native init app1
```

Then, update the generated package.json by setting the new package name and adding the `@my-app/app1` dependency:

```
{
- "name": "app1",
+ "name": "@my-app/app1",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "android": "react-native run-android",
    "ios": "react-native run-ios",
    "lint": "eslint .",
    "start": "react-native start",
    "test": "jest"
  },
  "dependencies": {
+   "@my-app/shared": "*",
+   "@react-native-async-storage/async-storage": "^1.18.1", // <- use for ExampleComponent in `@my-app/shared`
    "react": "18.2.0",
    "react-native": "0.71.7"
  },
  "devDependencies": {
    "@babel/core": "^7.20.0",
    "@babel/preset-env": "^7.20.0",
    "@babel/runtime": "^7.20.0",
    "@react-native-community/eslint-config": "^3.2.0",
    "@tsconfig/react-native": "^2.0.2",
    "@types/jest": "^29.2.1",
    "@types/react": "^18.0.24",
    "@types/react-test-renderer": "^18.0.0",
    "babel-jest": "^29.2.1",
    "eslint": "^8.19.0",
    "jest": "^29.2.1",
    "metro-react-native-babel-preset": "0.73.9",
    "prettier": "^2.4.1",
    "react-native-monorepo-tools": "^1.2.1",
    "react-test-renderer": "18.2.0",
    "typescript": "4.8.4"
  },
  "jest": {
    "preset": "react-native"
  }
}
```

update `packages/app1/App.tsx` to use `ExampleComponent` in `@my-app/shared` instead of the app template shipped with React Native:

```
// packages/app1/App.tsx
import {ExampleComponent} from '@my-app/shared';

function App(): JSX.Element {
  return <ExampleComponent />;
}

export default App;
```

Next try to run app1 in ios

```
yarn install && cd ios && pod install
```

The command will fail with an error like this:

```
[!] Invalid `Podfile` file: cannot load such file -- /Users/me/workspace/react-native-universal-monorepo-js/packages/mobile/node_modules/@react-native-community/cli-platform-ios/native_modules.

 #  from /Users/me/workspace/react-native-universal-monorepo-js/packages/mobile/ios/Podfile:2
 #  -------------------------------------------
 #  require_relative '../node_modules/react-native/scripts/react_native_pods'
 >  require_relative '../node_modules/@react-native-community/cli-platform-ios/native_modules'
 #
 #  -------------------------------------------
```

why we got this error. As we explained in the `1. setup monorepo` in `nohoist`, by default Yarn Workspaces will install the dependencies of each package (shared, app1, etc.) in `<project-root>/node_modules (AKA “hoisting”)`.
This behavior doesn’t work well with React Native, because the native code located in `app1/ios` and `app1/android` in some cases references libraries from `app1/node_modules` instead of `<project-root>/node_modules`.
Luckily, we can opt-out of Yarn workspaces’ hoisting for specific libraries by adding them to the `nohoist` setting in the root `package.json`:

```
// my-app/package.json
{
  "name": "my-app",
  "version": "0.0.1",
  "private": true,
  "workspaces": {
    "packages": [
      "packages/*"
    ],
    "nohoist": [
      "**/react",
      "**/react-dom",
+     "**/react-native",
+     "**/react-native/**"
    ]
  }
}
 ```
Adding the libraries from the diff above should be enough to make an app bootstrapped with React Native 0.71.7 work correctly:

- `**/react-native` tells Yarn that the react-native library should not be hoisted.
- `**/react-native/**` tells Yarn that the all react-native’s dependencies (e.g., metro, react-native-cli, etc.) should not be hoisted.

>You can completely opt-out from hoisting on all libraries (e.g., with "nohoist": ["**/**"]), but I wouldn’t advise doing so unless you feel like maintaining the list of hoisted dependencies becomes a burden.

Once you’ve updated the nohoist list, run yarn reset && yarn from the project root to re-install the dependencies using the updated settings.

Now try to `pod install` again it should be work.

### 3.1 setup metro bundler compatible with Yarn workspaces in app1

Before we can run the app, we still need do one more thing: make metro bundler compatible with Yarn workspaces’ hoisting.

Metro bundler is the JavaScript bundler currently used by React Native.

One of metro’s most famous limitations is its inability to follow symlinks.

Therefore, since all hoisted libraries (basically all libraries not specified in the nohoist list) are installed in `app1/node_modules` as symlinks from `<root>/node_modules`, metro won’t be able to detect them.
Additionally, <b>because of this issue, metro won’t even be able to resolve other workspaces (e.g., `@my-app/shared`) since they’re outside of the app1 directory.</b>

For example, running the app on iOS will now show the following (or a similar) error:

```
error: Error: Unable to resolve module @babel/runtime/helpers/interopRequireDefault from /Users/me/workspace/react-native-universal-monorepo-js/packages/app1/index.js: @babel/runtime/helpers/interopRequireDefault could not be found within the project or in these directories:
  node_modules

```

In this specific case, metro is telling us that he’s unable to find the `@babel/runtime` library in `app1/node_modules`. And rightfully so: `@babel/runtime` is not part of our `nohoist` list, so it will probably be installed in `<root>/node_modules` instead of `app1/node_modules`.


Luckily, we have [metro configuration options at our disposal](https://facebook.github.io/metro/docs/configuration/several) to fix this problem.

With the help of a couple of tools, <b>we can update the metro configuration file `(app1/metro.config.js)` to make metro aware of `node_modules` directories available outside of the app1 directory (so that it can resolve `@my-app/shared`)… with the caveat that libraries from the `nohoist` list should always be resolved from `app1/node_modules`.</b>

To do so, install [react-native-monorepo-tools](https://github.com/mmazzarolo/react-native-monorepo-tools), a set of utilities for making metro compatible with Yarn workspaces based on our `nohoist` list.

```
yarn add -D react-native-monorepo-tools
```

And update the metro config:

```
// my-app/packages/app1/metro.config.js
const exclusionList = require("metro-config/src/defaults/exclusionList");
const { getMetroTools } = require("react-native-monorepo-tools");

+ const monorepoMetroTools = getMetroTools();

module.exports = {
  transformer: {
    getTransformOptions: async () => ({
      transform: {
        experimentalImportSupport: false,
        inlineRequires: false,
      },
    }),
  },
+  // Add additional Yarn workspace package roots to the module map.
+  // This allows importing importing from all the project's packages.
+  watchFolders: monorepoMetroTools.watchFolders,
+  resolver: {
+    // Ensure we resolve nohoist libraries from this directory.
+    blockList: exclusionList(monorepoMetroTools.blockList),
+    extraNodeModules: monorepoMetroTools.extraNodeModules,
  },
}
```

Here’s how the new settings look like under-the-hood:

```
const path = require("path");
const exclusionList = require("metro-config/src/defaults/exclusionList");
const { getMetroTools } = require("react-native-monorepo-tools");

const monorepoMetroTools = getMetroTools();

module.exports = {
  transformer: {
    getTransformOptions: async () => ({
      transform: {
        experimentalImportSupport: false,
        inlineRequires: false,
      },
    }),
  },
  // Add additional Yarn workspaces to the module map.
  // This allows importing importing from all the project's packages.
  watchFolders: {
    '/Users/me/my-app/node_modules',
    '/Users/me/my-app/packages/app/',
    '/Users/me/my-app/packages/app2/',
    '/Users/me/my-app/packages/shared/'
  },
  resolver: {
    // Ensure we resolve nohoist libraries from this directory.
    // With "((?!app1).)", we're blocking all the cases were metro tries to
    // resolve nohoisted libraries from a directory that is not "app1".
    blockList: exclusionList([
      /^((?!app1).)*\/node_modules\/@react-native-community\/cli-platform-ios\/.*$/,
      /^((?!app1).)*\/node_modules\/@react-native-community\/cli-platform-android\/.*$/,
      /^((?!app1).)*\/node_modules\/hermes-engine\/.*$/,
      /^((?!app1).)*\/node_modules\/jsc-android\/.*$/,
      /^((?!app1).)*\/node_modules\/react\/.*$/,
      /^((?!app1).)*\/node_modules\/react-native\/.*$/,
      /^((?!app1).)*\/node_modules\/react-native-codegen\/.*$/,
    ]),
    extraNodeModules: {
      "@react-native-community/cli-platform-ios":
        "/Users/me/my-app/packages/app1/node_modules/@react-native-community/cli-platform-ios",
      "@react-native-community/cli-platform-android":
        "/Users/me/my-app/packages/app1/node_modules/@react-native-community/cli-platform-android",
      "hermes-engine":
        "/Users/me/my-app/packages/app1/node_modules/hermes-engine",
      "jsc-android":
        "/Users/me/my-app/packages/app1/node_modules/jsc-android",
      react: "/Users/me/my-app/packages/app1/node_modules/react",
      "react-native":
        "/Users/me/my-app/packages/app1/node_modules/react-native",
      "react-native-codegen":
        "/Users/me/my-app/packages/app1/node_modules/react-native-codegen",
    },
  },
}
```

Now let run app again it should work!

<b>Note</b> For android have assets resolution bug:

If you run your app on Android, you’ll notice that images won’t be loaded correctly:

This is because of [an open issue with the metro bundler logic used to load assets outside of the root directory on android](https://github.com/facebook/metro/issues/290) (like our app/src/logo.png image).

To fix this issue, we can patch the metro bundler assets resolution mechanism by adding a custom server middleware in the metro config.
The way the fix works is [quite weird](https://github.com/facebook/metro/issues/290#issuecomment-543746458), but since it’s available in `react-native-monorepo-tools` you shouldn’t have to worry too much about it.

You can add it to metro the metro config this way:

```
// my-app/packages/app1/metro.config.js

const exclusionList = require("metro-config/src/defaults/exclusionList");
const {
  getMetroTools,
+  getAndroidAssetsResolutionFix,
} = require("react-native-monorepo-tools");

const monorepoMetroTools = getMetroTools();

+ const androidAssetsResolutionFix = getMetroAndroidAssetsResolutionFix();

module.exports = {
  transformer: {
+    publicPath: androidAssetsResolutionFix.publicPath,
    getTransformOptions: async () => ({
      // Apply the Android assets resolution fix to the public path...
      transform: {
        experimentalImportSupport: false,
        inlineRequires: false,
      },
    }),
  },
+  server: {
+    // ...and to the server middleware.
+    enhanceMiddleware: (middleware) => {
+      return androidAssetsResolutionFix.applyMiddleware(middleware);
+    },
  },
  // Add additional Yarn workspace package roots to the module map.
  // This allows importing importing from all the project's packages.
  watchFolders: monorepoMetroTools.watchFolders,
  resolver: {
    // Ensure we resolve nohoist libraries from this directory.
    blockList: exclusionList(monorepoMetroTools.blockList),
    extraNodeModules: monorepoMetroTools.extraNodeModules,
  },
};
```

Try running Android — it should work correctly now

Developing and updating the app:

By using `react-native-monorepo-tools` in the metro bundler configuration, we are consolidating all our Yarn workspaces settings into the root `package.json`’s `nohoist` list.
Whenever we need to add a new library that doesn’t work well when hoisted (e.g., a native library), we can add it to the `nohoist` list and run `yarn` again so that the metro config can automatically pick up the updated settings.

Additionally, since we haven’t touched the native code, updating to newer versions of React Native shouldn’t be an issue (as long as there aren’t breaking changes in metro bundler).


Root-level scripts:
To improve a bit the developer experience, I recommend adding a few scripts to the top-level `package.json` to invoke workspace-specific scripts (to avoid having to cd into a directory every time you need to run a script).

For example, you can add the following scripts to the app1 workspace:
```
// my-app/packages/app1/package.json
"scripts": {
  "android": "react-native run-android",
  "ios": "react-native run-ios",
  "start": "react-native start",
  "studio": "studio android",
  "xcode": "xed ios"
},
```

And then you can reference them from the root this way:
```
// my-app/package.json
"scripts": {
   "reset": "find . -type dir -name node_modules | xargs rm -rf && rm -rf yarn.lock",
+  "app1:start": "yarn workspace @my-app/app1 start",
+  "app1:ios": "yarn workspace @my-app/app1 ios",
+  "app1:android": "yarn workspace @my-app/app1 android",
+  "app2:start": "yarn workspace @my-app/app2 start",
+  "app2:ios": "yarn workspace @my-app/app2 ios",
+  "app2:android": "yarn workspace @my-app/app2 android"
},

```
This pattern allows us to run workspace-specific script directly from the root directory.


Next, you can try to create workspace for another app in packages and follow step from `3. setup app1 workspace` 

## Installation this repo

clone project 

```
https://github.com/kittichanr/rn-monorepo-yarn-workspace.git
```

go to project and install dependencies

```
cd rn-monorepo-yarn-workspace && yarn install
```

go to app1 and app2 for pod install in IOS

```
cd packages/app1/ios && pod install
cd ..
cd packages/app2/ios && pod install
```

back to root directory and run app1 with this script:

```
yarn app1:start // yarn app2:start for app2
```

press `a` for build and run android app

press `i` for build and run ios app

