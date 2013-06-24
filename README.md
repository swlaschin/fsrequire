
## FsRequire - A dependency manager for single files ##

*DRAFT*

*This is currently just a proposal. Good idea? Stupid? Feedback welcome via issues!*

## Rationale ##

We are all familiar with NuGet, which is fine for managing entire assemblies at a binary level.

But with a concise language like F#, many useful libraries could be contained in *a single source file*. 

In which case, why not just include the file directly in your project, rather than adding yet another NuGet dependency?
This is especially true for smallish F# "wrapper" projects that wrap a object-oriented library with F# idioms. 

The advantages of this approach include:

* Reducing DLL dependencies. Library code is checked in to source control with all other code.
* Having the source directly available for [inspection](http://www.codinghorror.com/blog/2012/04/learn-to-read-the-source-luke.html), which also acts as way to demonstrate and share F# techniques.
* Makes it easy to have share personal or business "utility" libraries without needing to use NuGet at all.

## Proposal ##

The tool would perform the same tasks as any package manager:

* Installing (single file) "libraries" into your project.
* Automatically installing any dependencies as well.
* Automatically updating the library and the dependencies as newer versions become available.

The only difference is that this would be done at the source file level, rather than at the binary level.

This project is *not* trying to compete with NuGet or [NuGetPlus](http://mikehadlow.blogspot.co.uk/2013/06/guest-post-working-around-fnuget.html), which is clearly needed for large dependencies.
I think of them as complementary.  Yes, NuGet can manage content files as well, but that seems overkill for these single file cases.

A better model is something like [require.js](http://requirejs.org/) or [Node's module system](http://nodejs.org/api/modules.html). 

## Other Requirements ##

In addition to the basic requirements of any package manager, I propose some additional ones:

* *No dependencies*. The system should be able to be used with nothing other than F# (and FSI) installed.
* *Lightweight*. No databases, web services, etc. You should be able to "host" a local repository with zero setup.
* *Easy authoring*. It should be easy to author a library and share it with others on your team or the rest of the F# community.
* *Cross-platform*. Goes without saying.
* *Integrates with VCS*. The tool should work well in conjunction with a VCS such as Git/SVN.
* *Integrates with CI*. The tool should integrate with FAKE or other CI tools.
* *Plays well with others*. The tool should not affect the workings of other tools such as NuGet or Visual Studio.

## Examples ##

Examples of single file libraries that would be a good fit with FsRequire:

* **Logging**. A simple logging library, or a wrapper around log4net or similar.
* **Strings**. String manipulation and commonly used regexes.
* **Parsing**. Yaml and INI parsers, or grammars built on FParsec.
* **Command line parser** and associated helpers.
* **.NET wrappers**. Functional wrappers and helpers for .NET libraries and tools.
* **Database**. Lightweight database access such as [FsSql](https://github.com/mausch/FsSql).
* **Algorithms and data structures**. Useful functions and algorithms for collections, computation expressions, etc.

## Overview of FsRequire 

### The repository

All files are stored in a central git repository, which is cloned locally to act as the source.

In addition to the files, the repo contains a machine-readable index, and HTML catalogue page(s) listing all files by category, etc. 

### The files

Each file is standalone -- there are no packages, zip/tarballs, etc. 

Each file has a unique id, plus metadata such as platform constraints and other dependencies. FSI files are also supported.

### Using FsRequire in a project

To include a dependency in a project or directory, just create a special `.require` file listing the ids of the dependencies.

When `fsrequire` is run, for each config file detected, it will:

* Copy (or create links to) the required files, or update them if outdated,
* Copy (or create links to) any dependencies of the required files, or update them if outdated.
* Add entries to the project file for these files (if using VisualStudio/Xamarin/MonoDevelop) while preserving dependency order.

## Using FsRequire as a client

To use FsRequire to import existing libraries and their dependencies into your project, do the following steps.

### 1. Get the FsRequire repo 

Clone the FsRequire repo from GitHub to your local machine.

You can use the catalog.html page to view the contents.

### 2. Setup

To setup FsRequire for a particular solution or project, you can use FSI interactively, or use command line scripts.

In FSI do:

```
#load @"path/to/repo/fsrequire.fsx"
FsRequire.Setup("path/to/projects")
```

or from the command line:    

```
FSI /path/to/repo/fsrequire.fsx -setup path/to/projects 
```
    
This will:

* Create a `fsrequire.config` file under your home directory (for exact location, see below).
* An `_require` directory will be created under the solution root and under each project directory (if any).
* An initial ".require" file will be created in the _require directory.
* For project based systems (e.g. VisualStudio/Xamarin/MonoDevelop), the `_require` directory and config file will be added to the project file.

### 3. Editing the "requires" list

After FsRequire is installed, you can edit the `.require` file in a project to set up the required modules.

The content of a `.require` file is:

* a key:value pair, where the key is "fsrequire" and the value is a comma separated list of library ids

Example:

```
fsrequire: AModule, AnotherModule
```

To require .fsx files, use `fsxrequire` instead.

```
fsxrequire: AModule, AnotherModule
```

To constrain the platform, add the `platform` keyword:

```
platform: net40
fsrequire: AModule, AnotherModule
```


### 4. Updating the libraries

Once you have created the dependencies, you can run `fsrequire` to update them:

FSI /path/to/repo/fsrequire.fsx path/to/solution 

* `fsrequire` will detect all the .require files in the solution path and process them.
* For each config, all the requires will be parsed and looked up in the repository.
* The default repo is the location of `fsrequire.fsx`.
* All the depedendences of the requires will be extracted as well, to generate an update list.
* For these files, copy them from the repo to the project and create project links for them, or just update them if outdated. (see below for exact algorithm).


## Using FsRequire with source control and CI

When in comes to source control, there are two obvious options:

**1. Don't check in the libraries themselves, just the `.require` file.**

In this case, the libraries would be fetched from the local repo at build time, (if combined with a pull) would be guaranteed to be up to date.

**2. Do check in the source for the required libraries**

In this case, there is the possibility that the libraries are not up to date.

The libraries could be updated from the local repo at build time, as above, but this could mean that the commited source did not accurately reflect the latest update.

Instead, FsRequire should offer a "test for changes" option that would check for changes but not copy them over.
This could be used to force CI builds to fail if the committed libraries are not up to date.


## Using FsRequire as an author

### 1. Set your github login

This is to prevent you from accidentally adding files that do not have an author tag, or have a different author.

In FSI do:

```
#load @"path/to/repo/fsrequire.fsx"
FsRequire.SetAuthor("myname")
```

or from the command line:    

```
FSI /path/to/repo/fsrequire.fsx -setauthor "myname" 
```

This will update the `fsrequire.config` file in your home directory (for exact location, see below).

### 2. Add metadata to your file

The metadata is stored in the file itself, using the following format rules.

* The header is a multi-line comment starting with `(*fsrequire` and ending with `*)`. 
* Each metadata line inside the comment is of the form `key: value`. The colon-space is critical. A line without a colon-space will be ignored.
* The key must be a alphanumeric identifier. Key order is not important. If a key is duplicated, the last one is used.
* The value is everything from the colon-space to the end of line (therefore, no comments are allowed on the same line). The value is trimmed, and tabs are converted to spaces. 
* Certain keys are required, as documented in the "metadata" section later. If a file is used with FsRequire and does not have a FsRequire header, or is missing the required keys, the file will be rejected with errors.

Example of a file header:

```
(*===========================
My library 
===========================*)
(*fsrequire

id: XYZ.LoggingLib
title: XYZ's LoggingLib
framework: net40
authors: [ swlaschin ]
description: My logging library
dependencies: [ MyConfigLib, IniParser2 ]

*)

/// code starts here
module MyModule = 

```

### 3. Add/Update the local repository

In FSI do:

```
#load @"path/to/repo/fsrequire.fsx"
FsRequire.Add "path/to/file"
```

or from the command line:    

```
FSI /path/to/repo/fsrequire.fsx -add path/to/file
```

There are two situations:

* If the file id is new, the file is copied to the repo and the repo indexes updated.
* If the file id already exists, it will check the author tag against your login and warn you if they are not the same.

### 4. Push up to github and then send a pull request

* Fork the central repo
* Push your code up from your local repo
* Do a pull request to the central repo
* If the only changes are to files that have your github login in the "author" metadata, then the request will be automatically approved.
* Other pull requests will be moderated.

### 5. FSI files? 

FSI files are supported. Put the metadata in them and submit them as the main file as above. The sibling .fs file will be picked up and added to the repository automatically.

## Technical notes

The following sections are some technical notes for the proposal.

### Design decisions

There are a number of design decisions that are worth noting.

* **Centralized file-based repository**. This is modelled after the success of CPAN. 
  * With a central repository, it is easy to track cross dependencies such as "most depended on", age since last update, etc.
  * A file based repo means that you can trivially "host" a local repository. 
  * Obviously, if the file count gets too high, this decision will need to be revisited. That would be a nice problem to have!

* **No versioning**. In my view, versions add complexity, and are rarely used correctly, so I have left them out. 
  * In general, you want always want to upgrade to the latest version of a library, so a version is not needed.   
  * It's hard to manually create a version number. A correct way would be to analyze the public API and [use that for generating a version](http://raganwald.com/2013/06/20/bind-by-contract.html) -- but that is beyond the scope of this tool.
  * If there is a breaking API change, I suggest adding a version number to the id of the library instead. E.g. "XYZ.LoggingLib2".
  * If you need to stick with an older version for some reason, you just add the source file manually to your project and remove the dependency from FsRequire.

* **Documentation**. Documentation should be kept in the file along with the code. No separate files.
  * It can be a header comment, or using [literate F#](http://tpetricek.github.io/FSharp.Formatting/demo.html).
  * If you need more complicated documentation, the file is probably not a good candidate for FsRequest!

* **Installation scripts**. Many package managers have installation scripts that are run when a package is installed. FsRequire doesn't. It is a heavyweight feature and FsRequire is a lightweight system.
  * Simple installation scripts can be in the header and run manually.
  * For more complicated installation scripts, use a proper package manager!

* **Authentication of authors**. How do you maintain ownership so that no one can modify and resubmit your code? My solution is to piggyback on github authentication.
  * To submit a change to the central repository on GitHub, use a pull request.
  * If the only changes are to files that have your github login in the "author" metadata, then the request will be automatically approved.
  * Other pull requests will be moderated.

* **Security**. How can we ensure a file is correct and complete, and untampered with?
  * For the centralized repository, git does this for us. For local repositories, there is no anti-tamper guarantee.

* **Licensing**. All files in the repository will be licensed under Apache 2.0.
  * Rationale: it's the same license used for for F# and most other open source .NET projects.
  * Having to manage separate licenses for each library would be a nuisance.

* **Garbage collection**. To stop junk from accumulating, a simple garbage collection algorithm will be used: 
  * If a file has not be updated in a while (1 year, say) AND there are no dependents of the file, THEN it can be safely removed.
  * A popular but unchanged library can be prevented from being GC'd by being linked to a "root object" such as a "popular" list.
 
* **JSON for metadata**. The metadata is stored in JSON format rather than using F# code directly. 
  * Why? Because metadata can be easily parsed without needing to use FSI, e.g. using grep.

* **Test scripts**. It would be useful to have unit tests associated with each library as well.
  * A separate file should contains the tests, and the test file should have a dependency on the core file.
  * Issue: How to support different test frameworks?

### Metadata

Just as with NuGet, each "library" file will have a number of properties that identify it uniquely.

In NuGet, these properties are stored in a `.nuspec` file. In FsRequire, these properties can be stored in the file directly, as a header.  

The properties are the same as used by NuGet, except for `ext`, which is FsRequire specific.

* *id* (required). This uniquely identifies the library. As with NuGet, you should prefix your ids with an author or company namespace. E.g. "XYZ.LoggingLib".
* *ext* (required). A list of tags indicating whether it can be used as a .fsx script, or as .fs compiled into a project, or both. Options are "fs", "fsx".  Example: "ext: fs, fsx"
* *framework* (optional). Framework requirements, if any.  It uses the same conventions as NuGet, e.g. "net35". If absent, the file is assumed to support all .NET and mono frameworks higher than 2.0. 
* *title* (optional). E.g. "XYZ's Logging Lib". If absent, the `id` is used.
* *authors* (optional). E.g. "XYZ"
* *summary* (optional). E.g. "A useful logging library"
* *projectUrl* (optional). E.g. "http://example.com/LoggingLib/"
* *dependencies* (optional). The list of other FsRequire libraries it depends on, as a list of "id,version" pairs in JSON format. For example: `[ {id: xxx, version: 1.2.3}, {id: yyy, version: 4.5.6} ]`

### Library identifiers

A particular version of a library is uniquely described by three things: id; ext, and framework.

These can be conveniently encoded into the filename itself, in the form {id}_{ext}_{framework}.
  
### The repository index

To store all the available library files, we need some kind of repository. Since we are dealing with files, a full database is not needed, and a simple file based repository seems adequate.
So why not be on trend, and use Git as the grouping mechanism. That way, the files can be local.

Next, to determine the latest available version and dependencies for a given library, we require some sort of index into the repository.

For now, a simple flat file will suffice, I think!

* The file will be called `fsrequire.db`
* Each row in the file represents details for a particular version of a library.
* The data is stored using a standard format that is JSON compatible, namely a list of `key: value` pairs, wrapped in braces.
* The keys are from the metadata documented above.
* Example of a row with this data: 
  `{id: XYZ.LoggingLib, ext: fs, framework: net40, title: XYZ's LoggingLib, authors: XYZ, summary: ... dependencies: ...}`

### The repository catalogue

For now, a simple HTML file can be used as the human readable repository catalogue. It will be a static file, generated from scratch when files are added
(something like the [Hackage page](http://hackage.haskell.org/packages/archive/pkg-list.html) perhaps, but with a nicer skin and JS search).
  
### Installation algorithm

* For each required file, the index in the repo will be loaded to extract the dependencies
* A DAG will be created by recursing down the dependencies. Cycles will cause errors.
* For files in the DAG that are not in the project, an entry will be created in the project file. The project files will be created in dependency order.
* For files in the project that are not in the DAG, the entry will be removed from the project file. 
* When comparing a local file with a file in the repo, a simple hash will be used to test for differences.

### Location of per-user config file

* In Windows: under `%appdata%`. For example: `C:\Users\swlaschin\AppData\Roaming\FsRequire\config`
* In linux: under `~`. For example: `~/.fsrequire/config`

### Stable vs experimental branches

In some cases, you may want to deploy and/or consume a pre-release or unstable version of a library.  I propose using git branches for this.

* The "master" branch would be the stable one
* The "experimental" branch would be the place for unstable code. 
  * As an author, the deployment process is exactly the same
  * As an client, you would need to switch your local cloned repo to use the experimental branch
  * Alternatively, you could clone someone's fork rather than the central repo.

### Multiple repositories

In some cases, you might want to have two repositories, a personal/private one and the central one. 

This can be done by chaining the two so that the personal/private one is searched first, and then the central one. 

### Stats

It might be useful to allow the tool to send stats up to the cloud about what libraries it is using (opt-in only of course).
This would enable accurate analytics such as tracking popularity and frequency.
