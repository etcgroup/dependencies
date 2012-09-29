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
<project name="Demo" basedir="." default="dist">
	<include file="../dependencies/build.xml"/>

	<target name="deps">
		<echo>Collecting build tools into ${build.tools.dir}</echo>
		<antcall target="dependencies.build">
			<param name="target" value="jquery"/>
			<param name="dep.build.dir" value="${build.dir}"/>
		</antcall>
	</target>
	
	<target name="dist" depends="deps">
		<!-- do some other stuff -->
	</target>
</project>
```

First, we include the `build.xml` file from this project.
This sets up everything we need to be able to reference any dependency we might want.

A target is created for every dependency: for `jquery`, the target `dependencies.jquery`
is created.

However, these are not meant to be called directly. A `dependencies.build` target
is also created by `dependencies/build.xml`. In the example above, 
we use `<antcall target="dependencies.build">` to launch this target with some parameters.

The `target` parameter should be set to the name of the dependency we want to include.
This must be the same as the dependency's folder name.

The `dep.build.dir` parameter should be set to wherever you want the dependency to be
placed. In this case, we set it to `${build.dir}`, a property we defined elsewhere.

There are other parameters that you may want to use, which can vary from dependency to
dependency. For most, there is a `version` parameter:

```xml
<antcall target="dependencies.build">
	<param name="target" value="codeigniter"/>
	<param name="codeigniter.version" value="2.1-stable"/>
	<param name="dep.build.dir" value="${build.dir}/server"/>
</antcall>
```

This will cause the 2.1-stable branch of CodeIgniter to be loaded into the `${build.dir}/server` directory.


Process dependencies in parallel
--------------------

Loading a bunch of dependencies can be slow. Luckily, they are all independent of one another.
This means that we can use Ant's `<parallel>` task to load a bunch of dependencies at the same time.


Caching
--------

Since we are referencing publicly available downloads of compiled third-party libraries,
we may have to download a bunch of files before we can build. To avoid having to do this repetitively,
a `_cache` directory is created containing all of the downloaded files. 
If a file we need is already in the cache, it will not be downloaded again.

If for some reason something is wrong with your cache, you can simply delete it, or
use `ant dependencies.clear-cache`.


Adding a new dependency
--------------------

Adding a new dependency can be pretty simple, depending on how the specific
library in question is set up. It can also require some advanced Ant hacking.
We'll look at two examples


### Example 1: Backbone.js

Here are two examples. First, we'll discuss a simple one-file JavaScript library, Backbone.js.
Take a look at `/backbone/default.properties`:

```
backbone.version=0.9.2
backbone.url=https://raw.github.com/documentcloud/backbone/${backbone.version}/backbone-min.js
backbone.js.file=${dep.build.dir}/js/backbone.js
```

These properties are loaded during execution of the `dependencies.build` target, and 
can be overridden if they are set before that.

The `backbone.url` property is the url where `backbone-min.js` can be downloaded.
The `backbone.version` property is used to generate the appropriate url.
The `backbone.js.file` property can be helpful if your Ant script needs to know where backbone.js
ended up after you loaded the dependency. Note that in this case,
we are going to put the file in the `js` sub-directory.

Now let's look at `/backbone/build.xml`, where the real work gets done:

```xml
<project name="backbone">
	<target name="backbone">
	
		<get src="${backbone.url}" 
			dest="${dep.cache.dir}/backbone-${backbone.version}.js" skipexisting="true" verbose="true"/>
	
		<copy file="${dep.cache.dir}/backbone-${backbone.version}.js"
			tofile="${backbone.js.file}"/>
			
	</target>
</project>
```

It is important that the project name and target name both be `backbone`, the same as the folder name. 
This file makes use of the properties set in `default.properties`.

We first use Ant's `get` task to download the file to the cache directory.
By storing the downloaded file as `backbone-${backbone.version}.js` in the cache, we can cache
multiple versions of a dependency.

Next, we simply copy the downloaded file to wherever `backbone.js.file` points to.
That's all there is to it.


### Example 2: Rhino

Now let's look at a slightly more complicated example, which requires unpacking
a zip archive. Take a look at the files in `/rhino`.

The `default.properties` file for the Rhino dependency is pretty similar to the one
for Backbone, so we won't go into that. Let's look at Rhino's `build.xml`:

```xml
<project name="rhino">
	<target name="rhino">
		
		<get src="${rhino.url}"
			dest="${dep.cache.dir}/${rhino.zip.file}" skipexisting="true" verbose="true"/>
			
		<unzip src="${dep.cache.dir}/${rhino.zip.file}" dest="${dep.build.dir}">
			<patternset includes="**/${rhino.jar.filename}"/>
			<mapper type="flatten"/>
		</unzip>
		
	</target>
</project>
```

As with Backbone, we start by downloading the Rhino archive. Similarly, we store
the downloaded file as `rhino-${rhino.version}.zip` (`${rhino.jar.filename}` is set in 
`default.properties`, but this is just for convenience).

Next, we need to use the Ant `unzip` task to unzip the archive.
We tell unzip we will put the output in `${dep.build.dir}`, as with Backbone.

We have to use a `patternset` element to select which files within the zip archive
we actually are interested in (there are many others).
We also use a flatten `mapper` to pull the jar file out of the directory structure
inside the zip archive.

As before, the desired binary file will be placed in `${dep.build.dir}`.

When the structure of the zip archive gets more complicated, or
when there are multiple files to pull out (such as a css, js, and image files),
the `unzip` command in particular can get hairy. Check out `jqueryui`, `d3`, or `slickgrid` for examples.
