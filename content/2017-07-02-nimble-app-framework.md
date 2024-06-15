+++
title = "Hardware accelerated application framework in C++"
date = 2017-07-02
path = "2017/07/hardware-accelerated-application-framework"

[extra]
og_description = "I have decided to start working on a new hardware accelerated (cross-platform) application framework in C++, called Nimble Application Framework."
+++

I have decided to start working on a new hardware accelerated (cross-platform) application framework in C++, called Nimble Application Framework.

<!-- more -->

Nimble Writer has been the first product released under my company Nimble Tools. Unfortunately, it is not cross-platform, and it heavily relies on software painting with GDI. That means it will only work on Windows, and too many visual effects will cause it to flicker and/or be slow.

There has been an attempt to completely rewrite Nimble Writer from scratch using the (now seemingly discontinued) [Turbobadger library](https://github.com/fruxo/turbobadger). This is a library that provides a hardware accelerated widget framework with a full layout and render invalidation system. The idea is good, but it has a few things that are worth noting:

* It uses its own special file format with its own special syntax for defining styles and layouts. I've written a Sublime Text syntax highlighter for it, because there is nothing else for it.
* The code sometimes has issues when it comes to high DPI displays.
* It's not very easy to set up a new application with this system, eg. a skeleton project requires a bit too much code and configuration.
* It uses pure bitmaps for its styling instead of scalable graphics. (There's a branch on Turbobadger that provides this, but last time I checked it wasn't ready for production use.)

After using the library for a while, as well as submitting pull requests and maintaining a custom fork for my software purposes, I've decided to drop that library and finally start working on my own. Introducing: [Nimble Application Framework](https://github.com/nimbletools/nimble-app/).

This is an open source project similar to Turbobadger, but built from the ground up. Since the idea is to have a very simple API, the following is currently enough to show a window containing widgets defined in an xml file:

```cpp
#include <nimble/app.h>

class ExampleApplication : public na::Application
{
public:
	virtual void OnLoad()
	{
		LoadLayout("content/example.xml");
	}
};

int main()
{
	ExampleApplication* app = new ExampleApplication();
	app->Run();
	delete app;

	return 0;
}
```

As you can see, an application is basically a class inherited from `na::Application`. From there, you can load a layout XML file by calling `LoadLayout` with a filename. In this case, `example.xml` looks like this:

```xml
<hbox anchor="fill" color="#330000ff">
	<vbox anchor="left fillv" width="300" color="#003300ff" margin="5">
		<label fontsize="24" margin="5" font="content/Roboto.ttf">
			Nimble App Example
		</label>

		<button margin="5" font="content/Roboto.ttf">Create block</button>
		<button margin="5" font="content/Roboto.ttf">Invalidate layout</button>
		<button margin="5" font="content/Roboto.ttf">Resize window</button>
	</vbox>

	<hbox wrapping="true" anchor="fill" margin="5" />
</hbox>
```

For now I'm passing some styles that should not be required (font, margin) repeatedly. I will add a stylesheet system for this later, more on this in a future blogpost. If you run this example application, you will see this:

![](/2017/07/nimble-app-framework.png)

It's good to note that the application is rendered properly on my MacBook's high-DPI Retina screen. This means it will automatically adjust for whatever DPI the system is using.

Here are the key technologies and libraries I'm using to make all this happen:

* [GLFW](https://github.com/glfw/glfw) for the window creation, OpenGL context creation, and user input handling
* [NanoVG](https://github.com/memononen/nanovg) for primitive rendering, text rendering, and font loading
* [Layout](https://github.com/randrew/layout) for calculating widget rectangles
* [GLEW](https://github.com/nigels-com/glew) for OpenGL function bindings
* [GLM](https://github.com/g-truc/glm) for vector/matrix math
* [IrrXML](http://www.ambiera.com/irrxml/) for layout XML file parsing
* [Scratch2](https://github.com/codecat/scratch2) for the base classes (string, list, dict, …) to avoid having to rely on STL

Over the course of the next couple weeks I hope to be able to show more progress on this project, as well as some screenshots of the first application that will be developed using this new library. The library is fully open source and you can follow its development on [Github](https://github.com/nimbletools/nimble-app/).
