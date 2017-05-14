# How to Setup a Fable-Elmish Project (May 1 2017)

This is a short post that details the steps needed to setup a basic project with the following tools:

1. .NET Core and F#
1. Fable-Elmish
1. Working Time Travel Debugger on [Firefox](https://www.mozilla.org/en-US/firefox/) and [Chrome](https://www.google.com/chrome/).

At the end, you should have the sample Fable-Elmish project (showing navigation and counter example) running in a browser with a working time travel debugger.

These instructions are working as of 1 May 2017.

NOTE: If you are already familiar with Fable-Elmish, this post will probably not help you.  However, I would greatly appreciate any tips or corrections to improve these instructions.

# Setup .NET Core and F\#

The first step I took was to setup .NET Core.  I downloaded and installed 3 packages, in the given order.

## IDE

Install the IDE of your choice.  I decided to go with [Visual Studio Code](https://code.visualstudio.com/) and [Ionide](http://ionide.io/).

At a minimum, install the following extensions in VS Code to take full advantage of Ionide's F# development environment.
- Ionide-FAKE
- Ionide-fsharp
- Ionide-Paket

There are lots of additional extensions available in VS Code that can make your life easier during the development process.  Feel free to experiement!

## .NET Core

Install .NET Core from the [Microsoft Website](https://www.microsoft.com/net/download/core).
- Pick the appropriate version for your environment.
- I did this on a Windows 7 64-bit machine, and the x64 version worked correctly for me.
- At the time that I installed it, my version of the SDK was 1.0.3.

## Microsoft Build Tools

Install Microsoft Build Tools 2015 from the [Microsoft Website](https://www.microsoft.com/en-us/download/details.aspx?id=48159).

## F# 4.0

Install Microsoft Visual F# 4.0 Tools RTM from the [Microsoft Website](https://www.microsoft.com/en-us/download/details.aspx?id=48179).

At the time that I downloaded this, F# 4.1 was available.  However, I used F# 4.0 based on advice in an [issue](https://github.com/ionide/ionide-vscode-fsharp/issues/421) in the [Ionide GitHub project](https://github.com/ionide/ionide-vscode-fsharp).

# Setup Fable-Elmish

## Install `yarn`

[Yarn](https://yarnpkg.com/) is a [Node.js](https://nodejs.org/) package manager, similar to [`npm`](https://github.com/npm/npm).  Fable / Fable-Elmish are using `yarn`, so you will need to install it.

I already had [`Node`](https://nodejs.org/en/download/) and [`npm`](https://github.com/npm/npm) installed, so I installed `yarn` with  the following command.

```
npm install --global yarn
```

If you do not have `npm`, then you can download a `yarn` installer from the [Yarn Website](https://yarnpkg.com/en/docs/install).  Please note that I'm not sure if you will need `npm` as well, so keep in mind that you may need to install it later.

## Setup a new project with Fable-Elmish

Setting up Fable-Elmish in a way that you can use it for a new project requires a bit of CLI work.  Here are the steps.

1. Open a command prompt / shell.

1. Install the Fable-Elmish templates.  This is the command to install the templates for Fable-Elmish based on the [React framework](https://facebook.github.io/react/).
    ```
    dotnet new -i "Fable.Template.Elmish.React::*"
    ```

1. Create a new Fable-Elmish project.
    ```
    dotnet new fable-elmish-react -n project-name
    ```

1. Go to your new project directory.
    ```
    cd project-name
    ```

1. Install your Node-based dependencies.  Running `yarn` without parameters is equivalent to the `npm install` command.
    ```
    yarn
    ```

1. When I ran the commands, I received a warning from `yarn` about the version of my `remotedev` dependency.  When I looked at `package.json`, I saw that my `dependency` was at version `^0.2.7` but my `devDependency` was at version `^0.2.5`.
    - I fixed this situation by changing the `devDependency` version to `^0.2.7`.

1. Restore your NuGet-based dependencies.
    ```
    dotnet restore
    ```

# Setup debugging environment

## Chrome

1. Install the React Developer Tools from the [Chrome Web Store](https://chrome.google.com/webstore/detail/react-developer-tools/fmkadmapgofadopljbjfkapdkoienihi?hl=en).

1. Install the Redux DevTools from the [Chrome Web Store](https://chrome.google.com/webstore/detail/redux-devtools/lmhkpmbekcpmknklioeibfkpmmfibljd?hl=en).

## Firefox

1. Install the React Developer Tools from the [Mozilla Add-ons site](https://addons.mozilla.org/en-US/firefox/addon/react-devtools/?src=ss).  When installed, this add-on will be called React Devtools.

1. Install the Redux DevTools from the [Mozilla Add-ons site](https://addons.mozilla.org/en-US/firefox/addon/remotedev/?src=ss).

## Opera, IE, Safari, etc.

I have not tried these instructions myself, so please take with a grain of salt.

1. React Developer Tools offers a standalone version for browsers / environments other than Chrome and Firefox.  You can find the installation and usage instructions on the [react-devtools GitHub Website](https://github.com/facebook/react-devtools/tree/master/packages/react-devtools).

1. [zalmoxisus](https://github.com/zalmoxisus), the person who has released the Redux DevTools extensions for Chrome and Firefox, has a package called [`remote-redux-devtools`](http://zalmoxisus.github.io/monitoring/) for use with browsers / environments other than Chrome and Firefox.  You can find the installation and usage instructions on the [remote-redux-devtools GitHub website](https://github.com/zalmoxisus/remote-redux-devtools).

# Run the sample

1. To run the sample, go to the project directory and run the following command.
    ```
    dotnet fable npm-run start
    ```

1. Open your browser (I recommend Chrome or Firefox) and go to [http://localhost:8080/](http://localhost:8080/).

1. To open the debugger:
    - If you are using Chrome or Firefox, you will see a small green icon.  Clicking on it will bring up the Redux DevTools panel / window that will show state changes (if you've taken any actions in your browser).
    - If you decided to use the `remote-redux-devtools`, you may have you modify the default app.  If it doesn't work by default, see the [fable-elmish debugger GitHub Website](https://github.com/fable-elmish/debugger) for more details.

1. If you want to build the project for release, run the following command.
    ```
    dotnet fable npm-run build
    ```

# Next steps

At this point, the basics should be setup and the project is ready for your custom code.

If you are stuck, an excellent resource is the [Fable Gitter channel](https://gitter.im/fable-compiler/Fable), where a number of Fable and Fable-Elmish developers often hang out.

If you are not familiar with the [Elm architecture](https://guide.elm-lang.org/architecture/), that is a good next step.

This default version also does not give you certain often-requested features, such as using [Rollup](https://rollupjs.org/) instead of [Webpack](https://webpack.github.io/).

# Conclusion

If you are still reading, I hope this guide has helped you setup your first Fable-Elmish project.

If you have any suggestions, tips, or corrections, please leave them as a comment below.

See you next time!