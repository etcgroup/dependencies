Dependencies for ETC Group Software
===============

This repository contains references to all of the external dependencies
that we use in our own code.

In most cases, we just reference a URL where the dependency can be downloaded
in binary or compiled form.

In some cases, third party projects are not maintained in a way that works well with this system.
For example, there may not be stable links to builds for specific versions of the library.
We host our own copies of these binaries as downloads within the public-binaries project.
In general, we are trying to keep from managing third-party code and binaries as much as possible.

This has three main benefits:

1. We avoid having to store these potentially large files in our projects.
2. We don't have to manually pull in changes from third parties when they release updates.
3. By centralizing all of this information in one place, we can reference it
from any projects where we might use these libraries.


How to use
--------

The first step is to check out this repository using git.

Next, reference these dependencies from the Ant build script in your main project:

```xml
<project name="Demo" basedir="." default="build">
	<include file="../dependencies/build.xml"/>

	<target name="-get-deps" description="Get all the dependencies">

        <property name="build.dir" value="build"/>

		<echo>Collecting build tools into ${build.dir}</echo>

        <dependency name="jquery" dest="${build.dir}"/>
        <dependency name="backbone" dest="${build.dir}"/>
        <dependency name="underscore" dest="${build.dir}"/>
	</target>

	<target name="build" depends="-get-deps" description="Build the demo">
		<!-- do some other stuff -->
	</target>
</project>
```

First, we include `build.xml` file from the dependencies project.
This sets up everything we need to be able to reference any dependency we might want.

Specifically, a `dependency` macro is created that can be used to load dependencies
into a destination directory. For these elements, the `name` attribute should
be set to the name of the dependency we want to include. This must be the same
as the dependency's folder name in this project.

The `dest` attribute should be set to wherever you want the dependency to be
placed. In this case, we set it to `${build.dir}`, a property we defined above.

There are other parameters that you may want to use, which can vary from dependency to
dependency. For most, there is a `version` parameter, which can be set like this:

```xml
<dependency name="codeigniter" dest="${build.dir}">
    <property name="codeigniter.version" value="2.1-stable"/>
</dependency>
```

This will cause the 2.1-stable branch of CodeIgniter to be loaded into the `${build.dir}` directory.
You can look at how each dependency's properties file and build.xml work to see
what other parameters might be available to set.

Process dependencies in parallel
--------------------

Loading a bunch of dependencies can be slow. Luckily, they are typically independent of one another.
This means that we can use Ant's `<parallel>` task to load a bunch of dependencies at the same time.

```xml
<parallel>
    <dependency name="jquery" dest="${build.dir}"/>
    <dependency name="backbone" dest="${build.dir}"/>
    <dependency name="underscore" dest="${build.dir}"/>
</parallel>
```

Caching
--------

Since we are referencing third-party libraries, we have to download a bunch of
files before we can include them in our project. To avoid having to do this every
time you build, a `_cache` directory is created containing all of the downloaded files.
If a file we need is already in the cache, it will not be downloaded again.

If for some reason something is wrong with your cache, you can simply delete it, or
use `ant clear-cache` from within this project, or `ant dependencies.clear-cache`
from within the project importing dependencies.


Adding a new dependency
--------------------

Adding a new dependency can be pretty simple, depending on how the specific
library in question is set up. It can also require some advanced Ant hacking.
We'll look at two examples


### Example 1: Backbone.js

First, we'll discuss a simple one-file JavaScript library, Backbone.js.
Take a look at `/backbone/default.properties`:

```
backbone.version=0.9.2
backbone.url=https://raw.github.com/documentcloud/backbone/${backbone.version}/backbone.js
backbone.js.file=${dep.build.dir}/js/lib/backbone.js
```

These properties are loaded during your call to the `<dependencies>` macro, and
can be overridden if you provide them as properties in that macro.

The `backbone.url` property is the url where `backbone.js` can be downloaded.
The `backbone.version` property is used to generate the appropriate url for that version.
The `backbone.js.file` property reflects where backbone.js will end up, given dep.build.dir.
In this case, we are going to put the file in the `js/lib` sub-directory.

All of these properties are made available to your build script after you load the dependency.

Now let's look at `/backbone/build.xml`, where the real work gets done:

```xml
<project name="backbone">

    <target name="cache">
        <get src="${backbone.url}"
             dest="${dep.cache.dir}/backbone-${backbone.version}.js" skipexisting="true" verbose="true"/>
    </target>

    <target name="build">
        <copy file="${dep.cache.dir}/backbone-${backbone.version}.js"
              tofile="${backbone.js.file}"/>
    </target>
</project>
```

It is important that the dependency's project and folder name match (`backbone` in this case).

When you call `<dependency name="backbone" dest="${build.dir}"/>`, three things happen.
First, the properties file above is loaded, setting the url and destination for backbone.js.

Second, the `cache` target in this build.xml file is invoked, which uses Ant's `get`
task to download the file to the cache directory. By storing the downloaded file
as `backbone-${backbone.version}.js` in the cache, we can cache multiple versions
of a dependency at the same time. **Note**: the `skipexisting` attribute on `<get>`
means that the file will not be downloaded if it is present in the cache already.

Third, the `build` target is invoked, which simply copies the downloaded file
to wherever `backbone.js.file` points to. That's all there is to it.


### Example 2: Rhino

Now let's look at a slightly more complicated example, which requires unpacking
a zip archive. Take a look at the files in `/rhino`.

The `default.properties` file for the Rhino dependency is pretty similar to the one
for Backbone, so we won't go into that. Let's look at Rhino's `build.xml`:

```xml
<project name="rhino">

    <condition property="rhino.jar.exists">
        <available file="${rhino.jar.file}" />
    </condition>

    <target name="cache">
        <get src="${rhino.url}"
             dest="${dep.cache.dir}/${rhino.zip.file}" skipexisting="true" verbose="true"/>
    </target>

    <target name="build" unless="${rhino.jar.exists}">

        <unzip src="${dep.cache.dir}/${rhino.zip.file}" dest="${dep.build.dir}">
            <patternset includes="**/${rhino.jar.filename}"/>
            <mapper type="flatten"/>
        </unzip>

    </target>

</project>
```

As with Backbone, the `cache` target downloads the Rhino archive. Similarly, we store
the downloaded file as `rhino-${rhino.version}.zip` (`${rhino.jar.filename}` is set in
`default.properties`, but this is just for convenience).

In the `build` target, we need to use the Ant `unzip` task to unzip the archive.
We tell unzip we will put the output in `${dep.build.dir}`, as with Backbone.

We have to use a `patternset` element to select which files within the zip archive
we actually are interested in (there are many others).
We also use a flatten `mapper` to pull the jar file out of the directory structure
inside the zip archive.

Unpacking a zip file can take a few seconds, which can get pretty annoying when you
build often. To save time if possible, we set a `rhino.jar.exists` property when
the build.xml is loaded. This just checks if the rhino jar already exists in the
target location.

Then, on the `build` target, we use the `unless` attribute to say that we will not
bother unzipping the jar again if it is already present. This feature is convenient but
not essential.

When the structure of the zip archive gets more complicated, or
when there are multiple files to pull out (such as a css, js, and image files),
the `unzip` command in particular can get hairy.
Check out `jqueryui`, `d3`, or `slickgrid` for examples.

### Example 3: Converting JavaScript Libraries

Some JavaScript libraries do not support asynchronous module loading (AMD),
which we use to load javascript modules on our web pages.

In some cases, these must be converted so that they explicitly declare
what other modules they depend on. Otherwise the module loader may not load
them in the right order. Most often, this affects jQuery plugins,
which need to declare their dependence on jQuery.

You can configure a module to be converted in this way by adding a `module.files`
property to the default.properties file. See the example from slickgrid below:

```
slickgrid.version=2.0.2
slickgrid.url=https://github.com/mleibman/SlickGrid/zipball/${slickgrid.version}
slickgrid.js.file=${dep.build.dir}/js/lib/slick.core.js
slickgrid.css.file=${dep.build.dir}/css/slick.grid.css
slickgrid.module.files=${dep.build.dir}/js/lib/slick.grid.js
slickgrid.module.requires="'jquery', 'jquery.event.drag'"
```

The `module.files` property is a comma-separated list of JavaScript files that will
be individually converted into modules.

The `module.requires` property can be omitted if the only dependency is jQuery.
In this case, slickgrid also depends on jquery.event.drag, so we have added
that here.

Conversion surrounds the JavaScript in the targeted file with this wrapper:

```javascript
;(function() {
    "use strict";
    function __setup_wrapper__() {
    // The original file contents...
    }

    if (typeof define === 'function' && define.amd && define.amd.jQuery) {
        define([@{requires}], __setup_wrapper__);
    } else {
        __setup_wrapper__();
    }
})();
```

This allows the module to work as usual if used in a non-amd environment.