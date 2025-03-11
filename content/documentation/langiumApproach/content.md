+++
fragment = "content"
title = "Langium-Based DSL Approach"
weight = 150
[sidebar]
  sticky = true
+++

The Langium-Based DSL Approach is designed for modeling tools that are built around textual domain-specific languages. Leveraging Langium, it provides advanced textual language features including parsing, reference resolution, workspace indexing, and language server integration, but extends it with a modeling-tool-oriented API to simplify accessing the models via an API for non-textual editors.

<div style="text-align:center; margin-bottom:20px">
  <img src="../../images/modelingtools.svg" alt="Overview of Langium Approach" width="70%" />
</div>

## Core Features

- **Full Language Server Protocol support**: Including syntax highlighting, autocompletion, and validation
- **Advanced symbol and reference management**: Sophisticated reference resolution across files
- **DSL-specific model management**: Tailored to textual languages
- **Workspace indexing**: Efficient handling of large models across multiple files
- **Multi-client support**: Coordination across different editor instances

## When to Choose This Approach

The Langium-Based DSL approach is particularly suited for:

- Projects with a strong textual DSL component
- Large interconnected models that are spread across many files
- Scenarios requiring full control over the underlying textual format
- Applications where language features like autocompletion and validation are important

## Example Implementation

For an example application with this architecture, please refer to the CrossModel tool, available at:

https://github.com/CrossBreezeNL/crossmodel

## Diagram Editor
<div style="text-align:center; margin-top:50px; margin-bottom:50px">
  <img src="../../images/diagramanimated.gif" alt="Overview of GLSP with Langium" width="80%" />
</div>

### Introduction

Integrating diagram editors with Langium-based DSLs combines the power of graphical modeling with the precise control of textual DSLs. This approach leverages the [GLSP framework](https://eclipse.dev/glsp/) for diagram editing capabilities while using Langium for the underlying model representation and language services.

### Integration Architecture

When integrating GLSP diagram editors with Langium, the key components include:

1. **Language Server Integration**: The Langium language server provides model data and handles modifications to the textual representation
2. **Model Transformation**: Mechanisms to transform between the textual DSL representation and the graphical model
3. **Synchronization**: Keeping the diagram and text editor in sync when changes occur in either representation

### Example Implementation Guidelines

#### Setup and Configuration

To connect a GLSP diagram editor with a Langium-based DSL:

```typescript
// Configure language services integration
@inject(LangiumServices)
protected languageServices: LangiumServices;

// Setup document access
@inject(LangiumDocumentFactory)
protected documentFactory: LangiumDocumentFactory;
```

#### Model Loading

For loading the source model from Langium's textual representation:

```typescript
async loadSourceModel(action: RequestModelAction): Promise<void> {
  const modelUri = this.getUri(action);
  const document = await this.documentFactory.getOrCreateDocument(modelUri);
  const model = document.parseResult.value;
  
  // Store the model in GLSP's model state
  this.modelState.setSemanticRoot(modelUri, model);
  this.setupSynchronization(modelUri, document);
}
```

#### Synchronization with Text Changes

To synchronize the diagram when text changes occur:

```typescript
private setupSynchronization(modelUri: string, document: LangiumDocument): void {
  const workspace = this.languageServices.shared.workspace;
  
  workspace.onDidChangeTextDocument.event(event => {
    if (event.document.uri.toString() === modelUri) {
      // Update model state with new model
      const updatedModel = event.document.parseResult.value;
      this.modelState.setSemanticRoot(modelUri, updatedModel);
      
      // Trigger diagram update
      this.submissionHandler.submitModel().then(actions => {
        this.actionDispatcher.dispatchAll(actions);
      });
    }
  });
}
```

#### Converting Diagram Operations to Text Edits

When changes are made in the diagram, they need to be translated to appropriate text edits:

```typescript
async handleCreateNodeOperation(operation: CreateNodeOperation): Promise<void> {
  const document = await this.documentFactory.getOrCreateDocument(this.modelState.semanticUri);
  const textEdits: TextEdit[] = [];
  
  // Create appropriate text edit for the new node
  const nodeTextTemplate = this.generateNodeTextTemplate(operation);
  const insertPosition = this.determineInsertPosition(document);
  
  textEdits.push({
    range: insertPosition,
    newText: nodeTextTemplate
  });
  
  // Apply text edits via language server
  const workspace = this.languageServices.shared.workspace;
  await workspace.applyEdit({
    documentChanges: [{
      textDocument: { uri: document.uri.toString(), version: document.version },
      edits: textEdits
    }]
  });
}
```

These examples illustrate the general patterns for integrating GLSP diagram editors with Langium. The specific implementation details will vary depending on your DSL structure and diagram requirements.

For a complete reference implementation, please refer to the CrossModel project which successfully implements this integration pattern:

https://github.com/CrossBreezeNL/crossmodel

## Tree Editor
<div style="text-align:center; margin-bottom:20px">
  <img src="../../images/treeeditor.png" alt="Tree Editor Concept" width="80%" />
</div>

### Introduction

Tree-based editors provide hierarchical views of models backed by Langium DSLs. This approach combines the structured visualization benefits of tree editors with the precision and control of textual languages. Building tree editors for Langium-based DSLs involves connecting the [Theia Tree Editor](https://github.com/eclipse-emfcloud/theia-tree-editor) framework with Langium's language services.

### Integration Architecture

The integration between tree editors and Langium involves:

1. **Language Services Access**: Connecting to Langium language services to access parsed models
2. **Document Management**: Handling document changes and keeping the tree view synchronized
3. **Edit Operations**: Converting tree editor operations into appropriate text edits

### Implementation Guidelines

#### Tree Editor Configuration

First, set up the basic tree editor infrastructure with Langium service integration:

```typescript
@inject(LangiumServices) 
protected languageServices: LangiumServices;

@inject(LangiumDocumentProvider)
protected documentProvider: LangiumDocumentProvider;
```

#### Loading Models into the Tree

To load a Langium model into your tree editor:

```typescript
async init(): Promise<void> {
  // Initialize base tree editor
  await super.init();
  
  // Get the document from Langium
  const uri = this.getResourceUri()?.toString();
  if (uri) {
    const document = await this.documentProvider.getOrCreateDocument(uri);
    const model = document.parseResult.value;
    
    // Setup the tree with the model data
    this.updateTree(model);
    
    // Subscribe to document changes
    this.subscribeToModelChanges(uri);
  }
}
```

#### Synchronizing with Text Changes

Keep the tree editor in sync with changes to the underlying text document:

```typescript
private subscribeToModelChanges(uri: string): void {
  const workspace = this.languageServices.shared.workspace;
  
  workspace.onDidChangeTextDocument.event(event => {
    if (event.document.uri.toString() === uri) {
      // Update tree with new model when text changes
      const updatedModel = event.document.parseResult.value;
      this.updateTree(updatedModel);
    }
  });
}
```

#### Handling Tree Editor Updates

When users make changes in the tree editor, convert those changes to text edits:

```typescript
protected async handleTreeEdit(data: TreeEditor.EditData): Promise<void> {
  const uri = this.getResourceUri()?.toString();
  if (!uri) return;
  
  const document = await this.documentProvider.getOrCreateDocument(uri);
  const textEdits = this.convertToTextEdits(data, document);
  
  // Apply edits via language server
  const workspace = this.languageServices.shared.workspace;
  await workspace.applyEdit({
    documentChanges: [{
      textDocument: { uri, version: document.version },
      edits: textEdits
    }]
  });
}

private convertToTextEdits(data: TreeEditor.EditData, document: LangiumDocument): TextEdit[] {
  // Implementation depends on your specific DSL grammar
  // This maps tree edit operations to appropriate text changes
  // ...
}
```

The specific implementation of `convertToTextEdits` will vary based on your DSL structure and how tree operations should map to text changes.

### Conclusion

By following these patterns, you can create tree-based editors that provide structured views of your Langium DSL models while maintaining the benefits of a text-based representation. This approach leverages the strengths of both paradigms - giving users multiple ways to interact with the same underlying model.

For a complete reference implementation using a similar architecture, see the CrossModel project:

https://github.com/CrossBreezeNL/crossmodel

## Langium
To extend the Model Hub with your specific language, you need to provide a `ModelServiceContribution`.
A model service contribution can provide a dedicated Model Service API and extend the persistence and validation capabilities of the Model Hub.

One of the core principles in our Model Hub architecture is re-use and we therefore aim to re-use as much of the language infrastructure and support that Langium generates for us.
This is reflected in several design decisions:

- Since the language server that contains the modules with all the languages already starts in a dedicated process, we will start our own Model Hub server in the same process to ease access.

- We are re-using the dependency injection framework from Langium to bind our own Model Hub-specific services, such as the core model hub implementation, the model manager that holds the model storage, the overall command stack and the model subscriptions or the validation service that ensures that all custom validations are run on each model.

The bridge that connects the Model Hub world with the Langium world is our generic _EMF Cloud Model Hub Langium integration library_.
That library has two main components:

1. The Abstract Syntax Tree (AST) server that serves as a facade to access and update semantic models from the Langium language server as a non-LSP client.
It provides a simple open-request-update-save/close lifecycle for documents and their semantic model.

2. A converter between the Langium-based AST model and the client language model.

The biggest difference between those two models is that the client language model needs to be a serializable JSON model as we update the model using JSON patches and intend to send the model to clients which might run in a differenct process.
Please note that this JSON model is indepdendent from the serialization format that you define in your grammar from which the AST is derived.
This core work of the conversion is the resolution of cycles and the proper representation of cross references so that when we get a language model back from the client we can restore a full Langium-based AST model again, i.e., to have a full bi-directional transformation when it comes to the semantic model.

Using the generic AST server and the AST-language model converter we can easily implement a language-specific, typed `ModelServiceContribution` that the Model Hub can pick up and use as all we need to do is to connect our Langium services with the respective Model Hub services.
Any additional functionality that we want to expose for our language can be exported as Model Service API in the contribution and re-used in the model hub or even a dedicated server.

### Model Persistence

A model persistence contribution provides language-specific methods to load and store models from and to a persistent storage.
Based on the services generated by Langium, we can query the document storage from Langium and using the generic converter ensure that we return a serializable model for the Model Hub.
Similarly, we can re-use the generated Langium infrastructure to store the model by converting the language model back to an AST model.
Furthermore, we need to ensure that anytime a model is updated on the Langium side we properly update the model on the Model Hub side.
We achieve that by installing a listener on the Langium side and using the Model Manager from the Model Hub to execute a PATCH command that updates the model in the Model Hub.

For the Custom Model, the persistence contribution may look something like this:

```javascript

class CustomePersistence implements ModelPersistenceContribution<string, CustomModelRoot> {
  modelHub: ModelHub;
  modelManager: ModelManager<string>;

  constructor(private modelServer: CustomModelServer) {
  }

  async canHandle(modelId: string): Promise<boolean> {
    return modelId.endsWith('.custom');
  }

  async loadModel(modelId: string): Promise<CustomModelRoot> {
    const model = await this.modelServer.getModel(modelId);
    if (model === undefined) {
      throw new Error('Failed to load model: ' + modelId);
    }

    this.modelServer.onUpdate(modelId, async newModel => {
      try {
        // update model hub model
        const currentModel = await this.modelHub.getModel(modelId);
        const diff = compare(currentModel, newModel);
        if (diff.length === 0) {
          return;
        }
        const commandStack = this.modelManager.getCommandStack(modelId);
        const updateCommand = new PatchCommand('Update Derived Values', currentModel, diff);
        commandStack.execute(updateCommand);
      } catch (error) {
        console.error('Failed to synchronize model from CustomLanguageService', error);
      }
    });
    return model;
  }

  async saveModel(modelId: string, model: CustomModelRoot): Promise<boolean> {
    try {
      await this.modelServer.save(modelId, model);
    } catch (error) {
      console.error('Failed to save model' + modelId, error);
      return false;
    }
    return true;
  }  
}
```


### Model Validation

A model validation contribution can provide a set of validators that work on the semantic model of the Model Hub.
As a result, a validator can return a hierarchical diagnotic object that captures the infos, warnings, and errors of a particular part in the model.
Using the generic transformations between the Langium and Model Hub space, the main work in this contribution is the translation from Langium's `DiagnosticInfo` to the Model Hub's more generic `Diagnostic`.
Providing this translation as part of the generic _EMF Cloud Model Hub Langium integration library_ is on the roadmap but can be also extracted from any public example.

## DSL

[Langium](https://langium.org/) is an open source language engineering tool that allows you to declare the syntax of your modeling language in form of an EBNF-like grammar.
From that grammar, the Langium CLI can generate a complete TypeScript-based language server, including syntax highlighting, auto-completion, cross references, validation and many other features.
Internally, Langium uses a [Chevrotain](https://chevrotain.io/docs/) parser extended with an [ALL(*) algorithm](https://www.typefox.io/blog/allstar-lookahead) for unbounded lookahead and is re-using language server infrastructure classes from VS Code.
While Langium has more capabilities such as command line interface generation or visualization, we will focus on the aspects that relate to the model hub.

In Langium, a grammar is defined in a dedicated `.langium` file for which a VS Code Extension also provides tooling support.
Let's assume we want to have a simple grammar that follows a JSON syntax:

```ts
grammar CustomLanguage                    // grammar name

entry CustomModelRoot:                    // entry rule for the parser, i.e., document root
    Machine | WorkflowConfig;             // sequence of valid tokens → abstract syntax

fragment IdentifiableFragment:            // re-usable fragment
    '"id"' ':' id=STRING;

TYPE_MACHINE returns string: '"Machine"'; // we use a type to ease distinction for parsing
Machine:
    '{'
        IdentifiableFragment
        ',' '"name"' ':' name=STRING      // keywords as inline terminals → conrecte syntax
        ',' '"type"' ':' type=TYPE_MACHINE
        (',' '"workflows"' ':' '['
            ((workflows+=Workflow) (',' workflows+=Workflow)*)?
        ']')?                             // optional workflow children
    '}';

TYPE_WORKFLOW returns string: '"Workflow"';
Workflow:
    '{'
        IdentifiableFragment
        ',' '"name"' ':' name=STRING
        ',' '"type"' ':' type=TYPE_WORKFLOW
    '}';

TYPE_WORKFLOW_CONFIG returns string: '"WorkflowConfig"';
WorkflowConfig:
    '{'
        '"machine"' ':' machine=[Machine:STRING] // reference a machine defined somewhere else
        ',' '"workflow"' ':' workflow=[Workflow:STRING];
        ',' '"type"' ':' type=TYPE_WORKFLOW_CONFIG
    '}';

hidden terminal WS: /\s+/;                // Ignore whitespaces during parsing
terminal STRING: /"[^"]*"/;               // JSON only supports double quoted strings
```

If you try out this grammar on the [Langium Playground](https://langium.org/playground/?grammar=OYJwhgthYgBAwgewGbIKZoDJgHbAK5jBqylnkUUD0Vsok0cOkaAUK2jgC4gCeCKdGgCyiACZoANgCVEiLgC5Ky5TVice-EPkklkiOFwAWJAA4wAzmhAAaWAEsAdGkd2xiAMb4IG2CDlcrOSy8gDcKmRqVgCO%2BJweJCiwAG5gkvZisFyIANacFrCASYSwYABGFjxgHlywFrzcYAAe7CGKQWRSaD7cFgDUALwAFMJVRvY4JAA%2BsADqBjnIkogA7kg4yPbAAJQAVOHsyODA3TUAkhLc9htlugBiRydKEaRqIGgAtPgWN3oPGu2kADkACIMsDAbBAQoIRl%2BgBlAAq0lOADkAOKhdgIgCaAAUAKIAfWEAEF4AAJVH4vxoLj4EA4AoVEDjYBKEEjDxjCbg8JqZYkL4kMBZXimEjZdRgKywMT2Crjar2RA4WD6ODmEAWVmsTnctAKAGQgDegKN5HOGiu9h%2B9yIJ3NZEBNghIOYPnBkOhsHdaHhSNRaMoajyvGWBjEBWlDhw6QmWWsEHGaQKxQ8Kre1RIdQazRUztdwK4YrQnqhEOL4v6OIJxLJlJR%2BMdpEGBchwPDIAWS2WFjL3sBAG0zc8W4NO92Vn1%2BnMu4sVltYK2XbAJ-PewNZ5PlrstgB%2BZuQgC6gP3o-PwdoiFMXGVzEkq-m69gXPskjEbxwRsBAF9AZjWBrIkZgAeWkABpW5MBAmYaTpBkmR4Vl2WBLd115Vg0J7Q1yEBU1D0tS5rlKO4-m4Q82zdFh%2BwhX1-WRdEKJXEFK1LV1vVY6s8WAsDIOgmZvz-ACgMJUCIKgmDCXgECUVuU4gzeeDGVqJC8BQrCVjWDZgAwjTVhVbScKdfD82BaBXx5diIXM-V%2BkHPVxgNREGLRI9YFeNB0E-BISlgGzHNlTzHMyCxEB8ZYTDedRJCsJjCzXHsaMfOcezsvSFGcwMjzi9tWKSziRLEvjJOk2T5ME-92DGMQLgTEAk3vWY4SUKgAB0%2BiocILxeWhTmAHADBICL7C4NALHMBICjEelWVgTVtTwVhRvq5MH0y9EWuBQcAD1gSPHZgU67r3NoAApOEZNgFVJH4Cx8FMUwDC4KbEHwEiSFieQ0BC1TgAsIA&content=N4KABBYEQJYCZQFzQLYE8D6KCGBjAFjAHYCmUANOJFEdimclAOoD2RcJATgGYCuANmACyeQqQpUIUAC5oADg2giCxMpUjQA7i04BrbvxaaAzkjABtSRrDBo8M1HQZteg0YrRa9BwCESx6TBWV0NNDxl5RWYdfVCoMABfKwBdECSQUA1HUVUHVnYuPkFlMTUrKBdY90Y-AKCYtzD1alkFPIbQgGE2bhgAcyg0kCA) you'll see what content can and cannot be parsed.

Specifically, you would expect to see content like this:
```json
{
    "id": "my_machine",
    "name": "Wonderful Machine",
    "type": "Machine",
    "workflows": [
        { "id": "my_workflow", "name": "Best Workflow", "type": "Workflow" }
    ]
}
```

and

```json
{
    "machine": "Wonderful Machine",
    "workflow": "Best Workflow",
    "type": "WorkflowConfig"
}
```

When Langium encounters such content, it will first parse the document, export symbols into the global index, compute the local scope for symbols, link cross references according to the scope, index the resolved cross references and then validate the document.

However, when you input the examples above with our grammar, you will notice that there are a few things that do not work as expected out of the box:

1. References are done based on the `name` property instead of `id`.
2. The workflow cannot properly be referenced as it is only a child of the workflow.
3. If we write multiple grammars, we may need to repeat our terminal rules, i.e., `WS` or `STRING`.

Luckily, one of core principles in Langium is customization.
The core of that customization principle is a dependency injection framework with a set of default modules and their service implementations that can be overwritten in one central place.
Specifically, you define modules where implementations are bound on to a specific property and then create a set of services out of them using the injection mechanism.
Each module has a chance to provide new services, i.e., by specifying new properties, or override existing services by using an existing property name but being used later in the chain of modules.

In order to solve our first problem, we therefore would need to re-bind the default `NameProvider` from Langium and ensure that it uses our `id` property instead of the `name` attribute:


```ts
export interface ModelServicesExtension {
  references: {
    /** override */ NameProvider: NameProvider;
  }
}

export type ModelLanguageServices = LangiumServices & ModelServicesExtension;

export function createMyLangModule(context: {
  shared: ModelLanguagesSharedServices;
}): Module<ModelLanguageServices, ModelServicesExtension> {
  return {
    references: {
      NameProvider: () => new QualifiedIdProvider()
    }
  };
}

// creating the services from the modules
const shared = inject(createDefaultSharedModule(context), …);
const myLanguage = inject(createDefaultModule({ shared }), createMyLangModule({ shared }), …);

// usage: our NameProvider will be lazily created on access
myLanguage.references.NameProvider.getName()
```

In Langium, modules and the generated services can be split into two categories:
- Shared modules and services that mostly relate to the infrastructure such as the language server or the document management and build system.
- Language-specific modules and services that only relate to a single language such as parsing, auto-completion, or validation.

For more details on the dependency injection system and the individual default implementations, we refer to the [Langium documentation](https://langium.org/docs/).

As we have seen in this section, defining a grammar in Langium is very straight-forward.
However, there are certain services and capabilities that may be very common and can be shared through custom modules without having to re-implement them everytime.
Using Model Hub, we therefore offer some modules to support you in the implementation of JSON-based languages and also provide shared services and classes to ease the integration into the overall Model Hub architecture.
There is some future work on our road map to support the generation of a JSON-based grammar and overall integration based on a set of Typescript interface that represent the semantic model to make the definition of your modeling language even more efficient.
Of course, you are not limited to JSON-based grammars and there are also plans to support a YAML-like syntax.
However, if you are very keen on the textual representation of your grammar, you will always be able to simply define your own Langium grammar.

To see how we can integrate your modeling language into the Model Hub, see the next section.
