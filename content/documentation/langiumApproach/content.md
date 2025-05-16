+++
fragment = "content"
title = "Langium-Based DSL Approach"
weight = 150
[sidebar]
  sticky = true
+++

# Unleashing the Power of Textual DSLs

The Langium-Based DSL Approach offers a sophisticated foundation for building modeling tools centered around textual domain-specific languages. This approach harnesses the advanced capabilities of Langium — providing professional-grade language features such as parsing, reference resolution, workspace indexing, and language server integration — while extending it with a purpose-built API designed specifically for modeling tools. The result is a seamless experience for developers and users working with models through both textual and visual interfaces.

<div style="text-align:center; margin-bottom:20px">
  <img src="../../images/OverviewLangiumApproach.svg" alt="Overview of Langium Approach" width="70%" />
</div>

## Core Features and Benefits

- **Complete Language Server Protocol Integration**: Enjoy a rich development experience with syntax highlighting, intelligent autocompletion, and real-time validation that guides users toward correct model definitions
- **Sophisticated Symbol and Reference Management**: Navigate complex model relationships effortlessly with cross-file reference resolution that maintains model integrity
- **Purpose-Built Model Management**: Access models programmatically through APIs specifically designed for DSL interaction
- **High-Performance Workspace Indexing**: Handle enterprise-scale models spanning hundreds of files with efficient indexing and search capabilities
- **Seamless Multi-Client Collaboration**: Enable coordinated model editing across multiple editor instances with automatic synchronization

## When to Choose This Approach

The Langium-Based DSL approach delivers exceptional value for:

- **Text-Centric Modeling Projects**: When your primary interface for model creation and editing is textual
- **Large, Interconnected Model Ecosystems**: When managing extensive models distributed across numerous files
- **Custom Format Requirements**: When you need complete control over the textual representation of your models
- **Developer Experience Focus**: When advanced language features like autocompletion, validation, and navigation are critical to user productivity

## Creating your DSL with Langium

[Langium](https://langium.org/) is a powerful open-source language engineering tool that transforms an EBNF-like grammar into a complete TypeScript-based language server with features like syntax highlighting, auto-completion, cross-references, and validation.

### Grammar Definition

In Langium, you define your grammar in a dedicated `.langium` file. Here's an example of a simple grammar with JSON-like syntax:

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

This grammar would parse JSON content like:

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

### Customizing Language Behavior

Langium's flexibility allows you to customize its behavior through a dependency injection framework. For example, if you want references to use the `id` property instead of `name`:

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

The Model Hub offers pre-built modules to support JSON-based languages and provides shared services to simplify integration into the overall architecture. Future enhancements will include support for generating JSON-based grammars from TypeScript interfaces, making the creation of modeling languages even more efficient.

## Langium Integration: The Engine Behind Your Models

To extend the Model Hub with your specific language, you need to provide a `ModelServiceContribution` that integrates your Langium-based DSL with the Model Hub ecosystem. This contribution can deliver a specialized Model Service API while enhancing the persistence and validation capabilities of the Model Hub.

### Leveraging Existing Infrastructure

A core principle of the Model Hub architecture is efficient reuse of existing components. This approach is evident in how we utilize Langium's infrastructure:

- **Unified Process Architecture**: Since the language server containing all language modules already runs in a dedicated process, we start our Model Hub server in the same process for seamless integration.

- **Shared Dependency Injection**: We reuse Langium's dependency injection framework to bind Model Hub-specific services, including the core model hub implementation, model manager, command stack, model subscriptions, and validation services.

### The EMF Cloud-Langium Bridge

The connection between the Model Hub ecosystem and Langium is our comprehensive **EMF Cloud Model Hub Langium integration library**, which consists of two primary components:

1. **AST Server**: Acts as a facade to access and update semantic models from the Langium language server as a non-LSP client, providing a straightforward document lifecycle management system (open-request-update-save/close).

2. **Model Converter**: Transforms between the Langium-based AST model and the client-facing JSON model, resolving cycles and handling cross-references to ensure complete bidirectional transformation.

By using these components, implementing a language-specific `ModelServiceContribution` becomes straightforward — simply connect your Langium services with the Model Hub services and expose any additional functionality through your Model Service API.

### Model Persistence Implementation

A model persistence contribution provides language-specific methods to load and store models. Here's an example implementation:

```javascript
class CustomPersistence implements ModelPersistenceContribution<string, CustomModelRoot> {
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

A model validation contribution provides validators that work on the semantic model, returning hierarchical diagnostic objects that capture information, warnings, and errors. The translation from Langium's `DiagnosticInfo` to the Model Hub's `Diagnostic` format is the main task in this contribution.

### Cross References

The ModelHub handles cross-references through the `AstLanguageModelConverter`, converting Langium `Reference` objects to serializable `ReferenceInfo` objects:

```ts
export class DefaultAstLanguageModelConverter implements AstLanguageModelConverter {
  ...
  protected replacer(_source: AstNode, key: string, value: unknown): unknown {
    ...
    if (isReference(value)) {
      return this.replaceReference(value);
    }
    return value;
  }

  protected replaceReference(value: Reference<AstNode>): client.ReferenceInfo {
    return value.$nodeDescription && value.ref
      ? {
          $documentUri: getDocument(value.ref).uri.toString(),
          $name: value.$nodeDescription.name,
          $path: value.$nodeDescription.path,
          $type: value.$nodeDescription.type
        }
      : {
          $refText: value.$refText,
          $error: value.error?.message ?? 'Could not resolve reference: ' + value.$refText
        };
  }

  protected reviveNodeReference(container: AstNode, reference: client.NodeReferenceInfo): Reference {
    const node = this.resolveClientReference(container, reference);
    return {
      $refText: reference.$name,
      $nodeDescription: {
        documentUri: URI.parse(reference.$documentUri),
        name: reference.$name,
        path: reference.$path,
        type: reference.$type
      },
      $refNode: node?.$cstNode
    };
  }

  protected resolveClientReference(container: AstNode, reference: client.NodeReferenceInfo): AstNode | undefined {
    const uri = URI.parse(reference.$documentUri);
    const root = uri ? this.documents.getOrCreateDocument(uri).parseResult.value : container;
    return this.getAstNodeLocator(root.$document?.uri)?.getAstNode(root, reference.$path);
  }
}
```

## Diagram Editor: Visualizing Models with GLSP
<div style="text-align:center; margin-top:50px; margin-bottom:50px">
  <img src="../../images/diagramanimated.gif" alt="Overview of GLSP with Langium" width="80%" />
</div>

### Powerful Model Visualization and Editing

The integration of diagram editors with Langium-based DSLs combines the intuitive nature of graphical modeling with the precision and expressiveness of textual DSLs. This powerful combination leverages the [Eclipse GLSP framework](https://eclipse.dev/glsp/) for sophisticated diagram editing while using Langium to manage the underlying model representation and provide comprehensive language services.

### Seamless Integration Architecture

A well-designed integration between GLSP diagram editors and Langium includes these key components:

1. **Unified Language Server**: The Langium language server provides the authoritative model data and manages modifications to the underlying textual representation
2. **Bidirectional Transformation**: Specialized components translate between the textual DSL structure (AST) and the graphical model representation
3. **Real-time Synchronization**: Changes in either the diagram or text editor are instantly reflected across all views of the model

### Implementation Guidelines with Code Examples

#### Establishing the Connection

Connect your GLSP diagram editor with a Langium-based DSL using dependency injection:

```typescript
// Configure language services integration
@inject(LangiumServices)
protected languageServices: LangiumServices;

// Setup document access
@inject(LangiumDocumentFactory)
protected documentFactory: LangiumDocumentFactory;
```

#### Loading and Presenting Models

Transform Langium's textual representation into a visual diagram:

```typescript
async loadSourceModel(action: RequestModelAction): Promise<void> {
  const modelUri = this.getUri(action);
  const model = await this.modelAPI.get(modelUri);
  
  // Store the model in GLSP's model state
  this.modelState.setSemanticRoot(modelUri, model);
  this.modelAPI.subscribe(modelUri, () => {
    this.actionDispatcher.dispatchAll(actions);
  });
}
```

For a complete implementation of this integration pattern, refer to the CrossModel project:

https://github.com/CrossBreezeNL/crossmodel

## Real-World Implementation

For a comprehensive example of this architecture in action, explore the CrossModel tool:

https://github.com/CrossBreezeNL/crossmodel

## Take the Next Step with Langium-Based DSLs

The Langium-Based DSL Approach combines the precision of textual languages with the power of visual modeling tools, delivering a comprehensive solution for domain-specific modeling. By integrating directly with the Model Hub, your DSLs become part of a larger modeling ecosystem while maintaining their unique capabilities.

Ready to build your modeling language? Explore our examples and documentation to get started with this powerful approach.