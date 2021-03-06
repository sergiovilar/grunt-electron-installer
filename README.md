# Electron Installer Grunt Plugin

[![Build status](https://ci.appveyor.com/api/projects/status/yd1ybqg3eq397i26/branch/master?svg=true)](https://ci.appveyor.com/project/kevinsawicki/grunt-electron-installer/branch/master)

Grunt plugin that builds Windows installers for
[Electron](https://github.com/atom/electron) apps using
[Squirrel](https://github.com/Squirrel/Squirrel.Windows).

## Installing

```sh
npm install --save-dev grunt-electron-installer
```

## Configuring

In your `Gruntfile.coffee` or `Gruntfile.js` add the following:

```js
grunt.loadNpmTasks('grunt-electron-installer')
```

Then assuming you have an Electron app built at the given `appDirectory`,
you can configure the installer task like so:

```js
'create-windows-installer': {
  appDirectory: '/tmp/build/my-app',
  outputDirectory: '/tmp/build/installer',
  authors: 'My App Inc.',
  exe: 'myapp.exe'
}
```

Then run `grunt create-windows-installer` and you will have an `.nupkg`, a
`RELEASES` file, and a `.exe` installer file in the `outputDirectory` folder.

There are several configuration settings supported:

| Config Name           | Required | Description |
| --------------------- | -------- | ----------- |
| `appDirectory`        | Yes      | The folder path of your Electron app |
| `outputDirectory`     | No       | The folder path to create the `.exe` installer in. Defaults to the `installer` folder at the project root. |
| `loadingGif`          | No       | The local path to a `.gif` file to display during install. |
| `authors`             | Yes      | The authors value for the nuget package metadata. Defaults to the `author` field from your app's package.json file when unspecified. |
| `owners`              | No       | The owners value for the nuget package metadata. Defaults to the `authors` field when unspecified. |
| `exe`                 | No       | The name of your app's main `.exe` file. This uses the `name` field in your app's package.json file with an added `.exe` extension when unspecified. |
| `description`         | No       | The description value for the nuget package metadata. Defaults to the `description` field from your app's package.json file when unspecified. |
| `version`             | No       | The version value for the nuget package metadata. Defaults to the `version` field from your app's package.json file when unspecified. |
| `title`               | No       | The title value for the nuget package metadata. Defaults to the `productName` field and then the `name` field from your app's package.json file when unspecified. |
| `certificateFile`     | No       | The path to an Authenticode Code Signing Certificate |
| `certificatePassword` | No       | The password to decrypt the certificate given in `certificateFile` |
| `signWithParams`      | No       | Params to pass to signtool.  Overrides `certificateFile` and `certificatePassword`. |
| `setupIcon`           | No       | The ICO file to use as the icon for the generated Setup.exe |
| `remoteReleases`      | No       | A URL to your existing updates. If given, these will be downloaded to create delta updates |

## Sign your installer or else bad things will happen

For development / internal use, creating installers without a signature is okay, but for a production app you need to sign your application. Internet Explorer's SmartScreen filter will block your app from being downloaded, and many anti-virus vendors will consider your app as malware unless you obtain a valid cert.

Any certificate valid for "Authenticode Code Signing" will work here, but if you get the right kind of code certificate, you can also opt-in to [Windows Error Reporting](http://en.wikipedia.org/wiki/Windows_Error_Reporting). [This MSDN page](http://msdn.microsoft.com/en-us/library/windows/hardware/hh801887.aspx) has the latest links on where to get a WER-compatible certificate. The "Standard Code Signing" certificate is sufficient for this purpose.

## Handling Squirrel Events

Squirrel will spawn your app with command line flags on first run, updates,
and uninstalls. it is **very** important that your app handle these events as _early_
as possible, and quit **immediately** after handling them. Squirrel will give your
app a short amount of time (~15sec) to apply these operations and quit.

You should handle these events in your app's `main` entry point with something 
such as:

```js
var app = require('app');

var handleStartupEvent = function() {
  if (process.platform !== 'win32') {
    return false;
  }

  var squirrelCommand = process.argv[1];
  switch (squirrelCommand) {
    case '--squirrel-install':
    case '--squirrel-updated':

      // Optionally do things such as:
      //
      // - Install desktop and start menu shortcuts
      // - Add your .exe to the PATH
      // - Write to the registry for things like file associations and
      //   explorer context menus

      // Always quit when done
      app.quit();

      return true;
    case '--squirrel-uninstall':
      // Undo anything you did in the --squirrel-install and
      // --squirrel-updated handlers

      // Always quit when done
      app.quit();

      return true;
    case '--squirrel-obsolete':
      // This is called on the outgoing version of your app before 
      // we update to the new version - it's the opposite of
      // --squirrel-updated
      app.quit();
      return true;
  }
};

if (handleStartupEvent()) {
  return;
}
```

## Checking for updates, downloading and installing

```coffeescript
ChildProcess = require 'child_process'
path = require 'path'

class AutoUpdater

  appFolder = path.resolve(process.execPath, '..')
  electronFolder = path.resolve(appFolder, '..')
  updateDotExe = path.join(electronFolder, 'Update.exe')

  @setFeedUrl: (@updateUrl) ->

  _spawn = (command, args, callback) ->
    stdout = ''

    try
      spawnedProcess = ChildProcess.spawn(command, args)
    catch error
      # Spawn can throw an error
      process.nextTick -> callback?(error, stdout)
      return

    spawnedProcess.stdout.on 'data', (data) -> stdout += data

    error = null
    spawnedProcess.on 'error', (processError) -> error ?= processError
    spawnedProcess.on 'close', (code, signal) ->
      error ?= new Error("Command failed: #{signal ? code}") if code isnt 0
      error?.code ?= code
      error?.stdout ?= stdout
      callback?(error, stdout)

  spawn = (args, callback) ->
    _spawn(updateDotExe, args, callback)

  quitAndInstall =() ->
    spawnUpdate ['--processStart', exeName], ->
    require('app').quit();    

  downloadUpdate = (callback) ->
    spawn ['--download', @updateUrl], (error, stdout) ->
      return callback(error) if error?

      try
        # Last line of output is the JSON details about the releases
        json = stdout.trim().split('\n').pop()
        update = JSON.parse(json)?.releasesToApply?.pop?()
      catch error
        error.stdout = stdout
        return callback(error)

      callback(null, update)

  installUpdate = (callback) ->
    spawn(['--update', @updateUrl], callback)  

  checkForUpdates: ->
    throw new Error('Update URL is not set') unless @updateUrl

    downloadUpdate (error, update) =>
      if error?
        return

      unless update?
        return

      installUpdate (error) =>
        if error?
          return

        quitAndInstall()
```
