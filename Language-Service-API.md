> For a overview of the general TypeScript compiler architecture and layering, see [[Architectural Overview]]

## Overview

A simple analogy for a language service object is a long-lived program, or the compilation context. 

## Design Goals

There are two main goals that the language service design tries to achieve:

1. **On demand processing:**
The language service is designed to allow quick response that scales with the size of the program. The only way this can be achieved is only doing the absolute minimum work required. All language service interfaces only compute the needed level of information needed to answer the current query. For instance, a call to getSyntaxDiagnostics will only need the file in question to be parsed, but not type checked. A call getCompletionsAtPosition will only attempt to resolve declarations contributing to the type in question but not others.

2. **Decoupling compiler pipeline phases:** 
The language service design decouples different phases of the compiler pipeline, that would normally happen in order in one shot during command-line compilation; and it allows the language service host flexibility in ordering these different phases. 
For instance the language service reports diagnostics on a file per file bases, and for syntax errors of files separately from semantic errors, and form general options and resolution errors. This allows the host to create an optimized experience of getting syntax errors for the current file being edited quickly, without having to pay the cost of querying other files, or doing a full semantic check. It also allows the host to skip querying for syntax errors for files that have not changed. Similarly, the language service allows for emitting a file (getEmitOutput) without having to emit or even type check the whole program.

## Language Service Host

The host is descried by the LanguageServiceHost API, and it abstracts all interactions between the language service and the external world. The language service host defers managing, monitoring and maintaining input files to the host.

The language service will only ask the host for information as part of host calls. No asynchronous events, or background processing are expected. The host is expected to manage threading if needed.

The host is expected to supply the full set of files compromising the context. Refer to [reference resolution in the language service](https://github.com/Microsoft/TypeScript/wiki/Language-Service-API#reference-resolution-in-the-language-service) for more details.

## ScriptSnapshot
A ScriptSnapshot represents the state of the text of an input file to the language service at a given point of time. The ScriptSnapshot is mainly used to allow for an efficient incremental parsing. A ScriptSnapshot is meant to answer two questions, 1. get current text, and 2. given a previous snapshot what are the change ranges. The second is what incremental parsing uses mainly to ensure it only parses only the changed sections.

> For users who do not want to opt into incremental parsing, use `ts.ScriptSnapshot.fromString()`.

## Reference resolution in the language service
There are two two means of declaring dependencies in TypeScript, import statements, and triple-slash reference comments (/// <reference path="" />). Reference resolution for a program is the process of walking the dependency graph between files, and generating a sorted list of files compromising the program. 

In the command-line compiler (tsc) this happens as part of building the program. A createProgram call starts with a set of root files, parse them in order, and walk their dependency declaration (imports and triple-slash references) resolving references to actual files on disk and then pulling them into the compilation process.

This work flow is decoupled in the language service into two phases, allowing the host to interject at any point and change the resolution logic if needed. It also allows the host to fully manage program structure and optimize file state change.

> To resolve references originating from a file, use `ts.preProcessFile`. This method will resolve both imports and triple-slash references from this specific files. Also worth noting that this relies on the scanner, and does not require a full parse to allow for fast context resolution, suited to editor interactions.

A thing that is currently missing, and should be added in next release, is a built-in way to resolve imports. ts.preProcessFile returns import text, and the caller needs to resolve that to a .ts file path. [Issue #1793](https://github.com/Microsoft/TypeScript/issues/1793) tracks fixing it.

## Document Registry

A language service object corresponds to a single project. So if the host is handling multiple projects it will need to maintain multiple instances of the LanguageService objects; each instance of the language service holds state about the files in the context. Most of the state that a language service object holds is syntactic (text + AST). The projects can share files (at minimum, the library file lib.d.ts). The document registry is simply a store of SourceFile objects. If multiple language service instances share the same DocumentRegistry instance they will be able to share sourceFile objects if possible allowing for more efficient memory utilization.

A more advanced use of the document registry is to serialize sourceFile objects to disk and re-hydrate them when needed.

> The Language service comes with a default DocumentRegistry implementation allowing for sharing SoruceFiles between different LanguageService instance, use `createDocumentRegistry` to create one, and pass it to all your `createLanguageService` calls.