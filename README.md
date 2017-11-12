
Source is a native JS build system without external dependencies or the requirement for a console program, while also being a file managment system with module architecture. All instructions are handled by the
manifest, by passing query parameters or the in-built backend with UI and preview options.


It provides a unified workflow for developing in different environment, such as in the web, in worker and NodeJS. The path handling and file loading is the same everywhere.


## Getting started

Source can be used hybrid, for building and production as dynamic module loader. For using in the browser only source and the main file is required.

```
<html>
	<head>
		<script src="pathToSource/source.js"></script>
		<script src="index.js"></script>
	</head>
</html>
```

## Creating a context

Source creates contexts with manifest objects. Included files are migrated into the context object. A simple minimal context that inlcudes a stylesheet and a include for example:


```
Source.create({

    name: 'My App',
    
    includes: [
        'network.js',
        '/styles/style.css'
    ],

    main: function(app, Network) {

        // First parameter is the context 

    }
});
```

More complex manifest with explanation:

```
Source.create({

   name: 'My App',

   build: {

        container: 'HTML',
		
        stack: [
            {
                name: 'bundle'
            },
            {
                name: 'cc',
                enabled: false,
                flags: {
                    compilationLevel: 'SIMPLE'
                }
            }
        ]

    },

    // A context can have 3 include stages, can be globally predefined in a config.
    // Not all might be required, depending on the structure of the project.
    
    // Optional: globally required files
    
    use: [
    	
	// Inlcudes jquery from source-directory, fetches from window / global, prevents
	// minification, only in HTML environment
	
    	'@jquery.js $ -min -only:HTML'
    ],
    
    // Dependencies or child-contexts
    requires: [
        '@libraries/three/examples/js/libs/stats.min.js -i',
        '@vendor/mevedia/se/surreal-engine.js'
    ],

    // Actual includes, at this point these includes can access the context and all it's dependencies
    includes: [

        '@vendor/mevedia/common/network.js',
        '@vendor/mevedia/ui/workspace.js',
        '@vendor/mevedia/ui/animation.css'
    ],

   main: function(app, SE, Network, Workspace) {
   
   // First parameter is the context 

   }

});
```

#### Includes

Includes are module files. The context they are included from will be passed as last parameter. The module function can either monkey patch the context or return
a object. If a `displayName` property is provided, it will be appended to the context with this name.


```
Source.exports([

    'memory.js'

], function(Memory, context) {

	// One way
	
	context.doThings = function() {
	
	};

	
	// Another way
	
	return {
		displayName: 'MyLib'
	}
});
```

#### Query parameters

If the parameter `source` is in the URL, the application will load in the backend if no value is given. 

* No value

Project loads with backend

* build

Project builds and outputs as declared in the manifest (download by default)

* run

Project builds and executes with the final build


## UI Backend

A tree structure of all includes and their includes will be shown. The manifest can be edited and the build process can be executed with a preview of the final bundled file. All process information is shown in the console.

Plugins can be made for different stages, such as collection, processing, generating and any tasks in between. All tasks can be done asynchronous in the queue, for example the closure compiler is threaded. Examples can be found in `build/stack`.


Calling the project with `source` parameter without value (like http://localhost/myproject?source) it will load the project with backend

![alt text](https://mevedia.com/img/source_backend3.png "UI")

The compiled project can be previewed.

![alt text](https://mevedia.com/img/source_backend1.png "Preview")




#### Build options


```
Source.create({

	name: 'My App',
	
	build: {

		// If declared, explicitly only compatible with given container
		container: 'HTML',
		
		// Prefered output method by default 'download', the 'post' plugin can be used
		// for example to post the compiled source to a url.
		output: 'download',

		// Prepend Source for standalone, if Source isn't used globally. Default: true
		bootstrap: true,

		// Build stack, plugins are located in "build/stack/<name>/<name>.js"
		stack: [
		    {
			name: 'bundle'
		    },
		    {
			name: 'cc',
			enabled: false,
			flags: {
			    compilationLevel: 'SIMPLE'
			}
		    }
		]

    	}

});
```

#### Context events

A context can be hooked with different events.

- invoke

When a context has completed, this event is fired right before `main` is called. Includes get the context passed as last parameter and execute tasks when everything is loaded, but before `main` is called.

- loaded

Once all includes are loaded, this event is fired. Another list of files can be added now as if they were declared as includes in the manifest. The event is repeated as long as files get appended at this point.



#### Create a Worker

A worker will be loaded with source and handled as a file same as being in the main-thread.


```
Source.exports(function(app) {

	// Load 'thread.js', in the same directory as this file
	
	const thread = this.worker('/thread.js', thread => {
	
		thread.onmessage = function(e) {};
	
	});
	
});
```



### Nesting

A context can include files and other contexts. When nesting a context, options declared in the parent-context will be passed through, so
a child-context can receive options at inclusion time. One scenario is to pass a list of asynchronous things, that will be done before a deeper included file completes.




## Paths



### Import from global object / window

Handled for all supported containers. The global object is available as `Source.global`. By appending the name of the object after the path, like:
`jquery.js jQuery`



### Flags

`!<fileextension>`
Enforce file to load "as" (example: "style.css !txt")

`-priority:<number>`
**not used yet**

`-min`
Exclude from minifier

`-only:<container>`
File is only included in given container (example: "audiolib.js -only:HTML")


`-container:<container>:<alternateURL>`
Use different path if container matches the declared one

`-skip`
The include with this flag won't be passed as parameter.



### Custom loader

```
Source.Loader.add({
	type: 'asset',						// Type of file
	extensions: ['asset', 'pkg'],		// All file extensions handled by this loader
	containers: {
		HTML: {
			import: function(file) {
				
				// < Handle file loading asynchronous here >
				
				file.complete(); // Completes the file
				
			}
		},
		Worker: 'HTML'	// If the loading method is the same as another
	}
});
```


### Internals

Source provides some helpful functions out of box.


The `Source.PATH` object contains several path strings:

```
HOME - Path of the directory were `source.js` is loaded from
MAIN - Path of the directory the project main-context file is loaded from
ROOT - Relative path from the hostname, default is just '/' (optional)
BASE - Full url + ROOT
```

##### Constants

`Source.CONTAINER`
The current detected container, such as 'HTML', 'Node' or 'Worker'.


`Source.isMobile`
If running in a mobile device.

`Source.isHTML`
If the current context is a main-thread with all HTML5 features.

`Source.isWorker`
If the current context is in a worker.

`Source.isNode`
If the current context is in NodeJS.


##### Functions

`Source.filename(url)`
Returns the file name with extension.

`Source.pathname(url)`
Returns the path without filename.

`Source.argument(queryParameterName)`
Returns the value of a query parameter from the URL.

`Source.download(filename, data)`
Starts to download a text, buffer or a blob. Text will be converted into a blob for huge strings.

`Source.stringify(object)`
Turns a JS object into it's source code string version. Functions are supported too.

