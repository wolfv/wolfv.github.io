---
layout: post
title: "2020 ipywidgets workshop in Paris"
author: "Wolf Vollprecht"
summary: I have worked on finishing drag'n'drop in ipywidgets, and adding dynamic loading to JuppyterLab extensions. 
categories: posts
---

We've had five days of ipywidgets workshop in Paris, and while others worked on doing the hard work (fixing bugs and preparing the next big release of ipywidgets 8.0) I was finishing up an old PR for a really cool new feature (drag and drop!) and working on JupyterLab dynamic extension loading support for the upcoming JupyterLab 2.0!

![ipywidgets sprint 2020](/assets/images/2020/ipywidgets_sprint.jpg)

## Drag and Drop in ipywidgets

During the past summer, we've worked with [Bartosz Telenczuk](https://github.com/btel) on adding drag'n'drop support to ipywidgets. This work went very far, and there was an open pull request with all the good ideas inside. The initial work was done, but some cleanup and last bits of work before this could be merged. This last bit of work is now done.

<figure>
<video controls="true" autoplay loop width="100%">
	<source src="/assets/images/2020/drag_drop_ipywidgets.mp4" type="video/mp4" />
	<source src="/assets/images/2020/drag_drop_ipywidgets.webm" type="video/webm" />
</video>
<figcaption>The drag and drop example notebook</figcaption>
</figure>

The initial implementation was a draggable Label that could be dropped on containers. After we started with an implementation of a draggable `Label`, we settled on more generic widgets: a `DraggableBox` and a `DropBox`. Both are containers that can hold exactly one child. If the `draggable` trait on the `DraggableBox` is set to true, it can be dragged, and dropped, for example on a `DropBox`. One can install an `on_drop` handler on a  `DropBox` to receive data that was attached to the `DraggableBox`, as well as the widget that was dropped. That makes it possible to drag widgets from one container to another (as shown in the screencast). I think drag'n'drop will give a lot of power to the user, and I can see quite a few interesting applications for it, such as interactive plot builder, image sorting or dropping pins on a ipyleaflet map.

It is even possible to receive drag and drop events from the rest of the operating system, so one can also drag a file on top of a DropBox, and the widget will receive the mime type info, e.g. for uploading one or multiple images. Additioanlly, this also works in JupyterLab, and with detached widgets, as Jeremy Tuloup shows in this GIF: 

![](https://user-images.githubusercontent.com/591645/72636740-03b41a80-3960-11ea-9319-1b2abe583571.gif)

Here is the open PR: [https://github.com/jupyter-widgets/ipywidgets/pull/2720](https://github.com/jupyter-widgets/ipywidgets/pull/2720) which should go in for 8.0.

## JupyterLab dynamic extension loading

JupyterLab has a really nice extension system. Extensions can change almost every part of the JupyterLab app -- however, there is a drawback currently. As extensions are so tightly integrated, they are usually _compiled_ into the JupyterLab JavaScript. This means that when installing a new JupyterLab extension, the user needs to have NodeJS and yarn installed, and during the installation process a lot of memory, bandwidth and CPU is used to download all required NPM packages and compile them into a small production bundle.
Another hurdle is that in some organizations, access to NPM is forbidden -- as it comes with some security implications. 

Since JavaScript is a dynamic language, it also has an `eval` method, and we can dynamically load and execute scripts from the internet, or a file. Back in the days of JQuery, there was a utility called `RequireJS` to dynamically load JavaScript. We're using this again to load and register JupyterLab extension dynamically, and it's still working great. 

For this work we started a repository on GitHub, and at the moment it's called [jupyterlab-dynext](https://github.com/wolfv/jupyterlab-dynext). It's a JupyterLab extension that dynamically loads, wait for it, more JupyterLab extensions. Dynamic JupyterLab extensions look almost like regular extensions, with the small difference that the `requires` key shouldn't contain Token instances, but rather plain strings. These plain strings are then matched against other registered plugins in JupyterLab, and the correct dependencies are injected. 

For example:

```js
return function()
{
  return {
    id: 'mydynamicplugin',
    autoStart: true,
    requires: ["@jupyterlab/apputils:ICommandPalette"],
    activate: function(app, palette) {
      console.log("Hello from a dynamically loaded plugin!");

      // We can use `.app` here to do something with the JupyterLab
      // app instance, such as adding a new command
      let commandID = "MySuperCoolDynamicCommand";
      let toggle = true;
      app.commands.addCommand(commandID, {
        label: 'My Super Cool Dynamic Command',
        isToggled: function() {
          return toggle;
        },
        execute: function() {
          console.log("Executed " + commandID);
          toggle = !toggle;
        }
      });

      palette.addItem({
        command: commandID,
        // make it appear right at the top!
        category: 'AAA',
        args: {}
      });
    }
  };
}
```


The activate function will always receive the running JupyterLab app instance as the first argument, the other arguments are dynamically injected depending on values in the `requires` list -- in this case we want to add a command to the command palette, and in order to do that we need the ICommandPalette instance.

The jupyterlab-dynext loader can currently load any given URL (for example, you can use a GitHub Gist URL), and you can try out a new extension by executing the "Load current file as extension command" while editing the file. Note that this trick currently works only once, as the plugin IDs are unique and reloading is currently not allowed.

<video width="100%" controls>
    <source src="/assets/images/2020/dynamic_jlab_loading.mp4" type="video/mp4" />
    <source src="/assets/images/2020/dynamic_jlab_loading.webm" type="video/webm" />
</video>

In the future it would be very cool to have a sort of snippet editor for these dynamically loaded extensions, with a big reload button to test extensions or themes quickly. One could also expose these JS loading capabilities as a widget to the notebook, in order to bring some of the old `%% javascript` magic back. If you are interested in hacking on this and currently are a student, don't hesitate to contact us as we're still looking for interns at QuantStack.

<figure>
	<img src="/assets/images/2020/mockup.png">
	<figcaption>Mockup of a extension manager/editor with reload</figcaption>
</figure>

#### JupyterLab Dark Theme

I am also still quite frustrated with the dark theme of JupyterLab. I think it was a hasty job back then, where colors were not correctly adjusted to make it look great. The black is very saturated, and some of the colors for syntax highlighting have a low contrast ratio. So I quickly ported a different dark theme to JupyterLab, mostly by using the style editor in Firefox: Turns out it's really simple to make themes for JLab, just installing them is as tedious as it is with any extension – something we're trying to fix with the dynamic extension loader as well. It should probably have a way to load a theme style sheet without executing any JavaScript so that installing themes doesn't have any security implications. But as it stands right now, it doesn't look like this (or any other better dark theme) will make the cut for JupyterLab 2.0 [(pull request is open here)](https://github.com/jupyterlab/jupyterlab/pull/7779).

![JupyterLab AYU theme](/assets/images/2020/jlab_color.jpg)

## Fixing up compilation time and binary size of xwidgets

The [xwidgets](https://github.com/jupyter-xeus/xwidgets) are the C++ counterpart to the ipywidgets. They support exactly the same functionality, but the backend is implemented in C++. The frontend, however, runs exactly the same JavaScript code in the webbrowser (that's the beauty of the widgets!).

However, we've been using some elaborate template meta-programming techniques in xwidgets which allow the code to look _quite similar_ to Python and supporting the same value semantics. This comes with a burden, though: for one,
compiled binaries are huge, and interactive execution via the xeus-cling just-in-time compiler is slow.

I have started to re-implement the core part of xwidgets, xproperty, from the ground up with some new ideas. This was partially successful, and inspired Sylvain and Johan to go back to xproperty and fix it up with the new ideas. Binary size is now cut in half, and compilation is also a little bit faster.

Still I think it could be interesting to implement xproperty in an even more dynamic way, getting rid of a lot of templates and relying fully on virtual functions.

--- 

All in all this was a very productive workshop. Many thanks to Jason for helping so much with the JupyterLab dynamic loading extension.

You can follow me on [↝ Twitter](https://twitter.com/@wuoulf) to stay up to date.