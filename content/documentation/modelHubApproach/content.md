+++
fragment = "content"
title = "Model Hub Approach"
weight = 160
[sidebar]
  sticky = true
+++

The Model Hub Approach delivers a powerful, model-first solution for enterprise-grade modeling applications. Built around a centralized Model Hub, it provides advanced capabilities including command-based editing, comprehensive state management with undo/redo functionality, and robust persistence — making it the ideal foundation for scenarios demanding precise control and model-oriented workflow.

<div style="text-align:center; margin-bottom:20px">
  <img src="../../images/OverviewModelHubApproach.svg" alt="Overview of the Model Hub" width="70%" />
</div>

### Core Features & Benefits

- **Command-based editing**: Implement complex model operations through a structured command system, ensuring data integrity
- **Centralized state management**: Maintain consistent model state with powerful undo/redo capabilities
- **Robust persistence layer**: Deploy flexible storage strategies adaptable to your business requirements
- **Real-time multi-client support**: Enable seamless collaboration with synchronized updates across all clients
- **Comprehensive model validation**: Catch errors early with the built-in validation framework

### When to Choose This Approach

The Model Hub approach delivers exceptional value for:

- Modeling tools requiring granular control over model changes
- Applications with complex editing operations needing comprehensive undo/redo capabilities
- Collaborative solutions where multiple users must work simultaneously
- Projects where structured model data (rather than plain text files) is your primary focus

### ModelHub: The Central Intelligence for Model Management

The EMF Cloud Model Hub serves as the central coordination hub for model management, orchestrating multiple clients (like different editors) and their interactions with models. Beyond providing a generic model access API, the Model Hub is highly extensible to support various modeling languages.

#### Interacting with models

Applications can contain multiple model hubs, each with a unique context identifier. This allows for flexible organization based on application requirements.

Access the model hub for a specific context using the ModelHubProvider:

```ts
import { ModelHubProvider } from '@eclipse-emfcloud/model-service-theia/lib/node/model-hub-provider';
import { ModelHub } from '@eclipse-emfcloud/model-service';

@injectable()
class ModelHubExample {
  @inject(ModelHubProvider)
  modelHubProvider: ModelHubProvider

  modelHub: ModelHub;

  async initializeModelHub() {
    this.modelHub = await modelHubProvider('my-application-context');
  }
}
```

##### Loading and saving models

Load models using the model hub's asynchronous API:

```ts
const modelId = 'file:///custom-editor/examples/workspace/my.custom';
const model: object = await modelHub.getModel(modelId);
```

With type safety:

```ts
const modelId = 'file:///custom-editor/examples/workspace/my.custom';
const model: CustomModelRoot = await modelHub.getModel<CustomModelRoot>(modelId);
```

Save models after changes:

```ts
// Using modelId as the commandStackId for single-model operations
const modelId = 'file:///custom-editor/examples/workspace/my.custom';
const commandStackId = modelId;
modelHub.save(commandStackId);
```

Or save all modified models:

```ts
modelHub.save();
```

##### Resolving references

ModelHub provides sophisticated handling of cross-references between model elements, representing them as structured information objects:

```ts
export interface NodeInfo {
  /** URI to the document containing the referenced element. */
  $documentUri: string;
  /** Navigation path inside the document */
  $path: string;
  /** `$type` property value */
  $type: string;
  /** Name of element */
  $name: string;
  /** Generic object properties */
  [x: string]: unknown;
}

export interface ReferenceError {
  $refText: string;
  $error: string;
}

export type ReferenceInfo = NodeReferenceInfo | ReferenceError;
```

On the client side, reference resolution is simplified with utility functions:

```ts
export type Reference<T> = Partial<NodeReferenceInfo> &
  Partial<ReferenceError> & {
    element(): Promise<T | undefined>;
    error(): string | undefined;
  };

export type ReferenceFactory<T> = (info: ReferenceInfo) => Reference<T>;

export function reviveReferences<T extends object>(obj: T, referenceFactory: ReferenceFactory<T>): T {
  for (const [key, value] of Object.entries(obj)) {
    if (isReferenceInfo(value)) {
      (obj as any)[key] = referenceFactory(value);
    } else if (value && typeof value === 'object') {
      reviveReferences(value, referenceFactory);
    }
  }
  return obj;
}
```

##### Changing models

Models are modified through specialized Model Services that implement domain-specific editing operations as Commands on a CommandStack:

```ts
export interface CustomModelService {
  getCustomModel(modelUri: string): Promise<CustomModelRoot | undefined>;

  unload(modelUri: string): Promise<void>;

  edit(modelUri: string, patch: Operation[]): Promise<PatchResult>;

  createNode(modelUri: string, parent: string | Workflow, args: CreateNodeArgs): Promise<PatchResult>;
}

export const CUSTOM_SERVICE_KEY = 'customModelService';

// Access and use the model service
const modelService: CustomModelService = modelHub.getModelService<CustomModelService>(CUSTOM_SERVICE_KEY);
await modelService.createNode(modelId, '/workflows/0', { type: 'AutomaticTask' });
```

##### Validating models

Model validation ensures data integrity through registered Validators:

```ts
const modelId = 'file:///custom-editor/examples/workspace/my.custom';
const diagnostic = await modelHub.validateModels(modelId);
```

Validate multiple models simultaneously:

```ts
const modelId1 = 'file:///custom-editor/examples/workspace/foo.custom';
const modelId2 = 'file:///custom-editor/examples/workspace/my.custom';
const diagnostic = await modelHub.validateModels(modelId1, modelId2);
```

Or validate all currently loaded models:

```ts
const diagnostic = await modelHub.validateModels();
```

For accessing the latest known validation results without triggering revalidation:

```ts
const modelId = 'file:///custom-editor/examples/workspace/my.custom';
const currentDiagnostic = modelHub.getValidationState(modelId);
```

#### Contributing modeling languages

Extend the ModelHub with custom modeling languages by implementing the `ModelServiceContribution` interface, which can handle:

- Persistence (Save/Load)
- Editing (Via Model Services and Commands)
- Validation
- Triggers

Here's a minimal example:

```ts
export const CUSTOM_SERVICE_KEY = 'customModelService';

@injectable()
export class CustomModelServiceContribution extends AbstractModelServiceContribution {
  private modelService: CustomModelService;

  @postConstruct()
  protected init(): void {
    this.initialize({
      id: CUSTOM_SERVICE_KEY
    })
  }

  getModelService<S>(): S {
    if (! this.modelService){
      this.modelService = new CustomModelServiceImpl();
    }
    return this.modelService as unknown as S;
  }
}
```

A more comprehensive implementation with persistence capabilities:

```ts
export const CUSTOM_SERVICE_KEY = 'customModelService';

@injectable()
export class CustomModelServiceContribution extends AbstractModelServiceContribution {

  private modelService: CustomLanguageModelService;

  constructor(@inject(CustomLanguageModelService) private languageService: CustomLanguageModelService){
    // Empty constructor
  }

  @postConstruct()
  protected init(): void {
    this.initialize({
      id: CUSTOM_SERVICE_KEY,
      persistenceContribution: new CustomPersistenceContribution(this.languageService)
    });
  }

  getModelService<S>(): S {
    return this.modelService as unknown as S;
  }

  setModelManager(modelManager: ModelManager): void {
    super.setModelManager(modelManager);
    // Forward the model manager to our model service
    this.modelService = new CustomModelServiceImpl(modelManager, this.languageService);
  }
}

class CustomPersistenceContribution implements ModelPersistenceContribution {
  
  constructor(private languageService: CustomLanguageModelService) {
    // Empty
  }

  canHandle(modelId: string): Promise<boolean> {
    // Handle file URIs with '.custom' extension
    return Promise.resolve(modelId.startsWith('file:/') 
      && modelId.endsWith('.custom'));
  }

  async loadModel(modelId: string): Promise<object> {
    // Load model from file...
  }

  async saveModel(modelId: string, model: object): Promise<boolean> {
    // Save model to file...
  }
}
```

##### Persistence

Register a `ModelPersistenceContribution` to handle model loading and saving:

```ts
@postConstruct()
protected init(): void {
  this.initialize({
    id: CUSTOM_SERVICE_KEY,
    persistenceContribution: new CustomPersistenceContribution(this.languageService)
  });
}
```

The contribution must implement three key methods:

```ts
class CustomPersistenceContribution implements ModelPersistenceContribution {
  
  constructor(private languageService: CustomLanguageModelService) {
    // Empty
  }

  canHandle(modelId: string): Promise<boolean> {
    // Handle file URIs with '.custom' extension
    return Promise.resolve(modelId.startsWith('file:/') 
      && modelId.endsWith('.custom'));
  }

  async loadModel(modelId: string): Promise<object> {
    // Load model from file...
  }

  async saveModel(modelId: string, model: object): Promise<boolean> {
    // Save model to file...
  }
}
```

##### Editing Domain

Model editing is handled through the ModelManager, which is accessed via Model Services that execute Commands on CommandStacks:

###### Model Service implementation

The ModelService uses the ModelManager to execute Commands that modify models:

```ts
export class CustomModelServiceImpl implements CustomModelService {
  constructor(private modelManager: ModelManager<string>) {}

  async getCustomModel(modelUri: string): Promise<CustomModelRoot | undefined> {
    const key = getModelKey(modelUri);
    return this.modelManager.getModel<CustomModelRoot>(key);
  }

  async createNode(modelUri: string, parent: string | Workflow, args: CreateNodeArgs): Promise<PatchResult> {
    const model = await this.getCustomModel(modelUri);
    if (model === undefined) {
      return {
        success: false,
        error: `Failed to edit ${modelUri.toString()}: Model not found`
      };
    }

    // Resolve parent element
    const parentPath = this.getParentPath(model, parent);
    const parentElement = getValueByPointer(model, parentPath);
    if (!isWorkflow(parentElement)) {
      throw new Error(`Parent element is not a Workflow: ${parentPath}`);
    }

    // Create new node
    const newNode = createNode(args.type, parentElement, args);

    // Create JSON patch
    const patch: Operation[] = [
      {
        op: 'add',
        path: `${parentPath}/nodes/-`,
        value: newNode
      }
    ];
    
    // Get command stack and execute command
    const stackId = getStackId(modelUri);
    const stack = this.modelManager.getCommandStack(stackId);
    const command = new PatchCommand('Create Node', modelUri, patch);
    const result = await stack.execute(command);

    // Return result
    const patchResult = result?.get(command);
    if (patchResult === undefined) {
      return {
        success: false,
        error: `Failed to edit ${modelUri.toString()}: Model edition failed`
      };
    }
    return {
      success: true,
      patch: patchResult
    };
  }
}
```

Alternative approaches for creating patch commands include using JSON Patch libraries:

```ts
// Using fast-json-patch to generate a diff
const updatedModel = deepClone(model) as CustomModelRoot;
const updatedWorkflow = getValueByPointer(updatedModel, parentPath);
updatedWorkflow.nodes.push(newNode);
const patch: Operation[] = compare(model, updatedModel);
const command = new PatchCommand('Create Node', modelUri, patch);
```

Or using the built-in Model Updater for direct editing:

```ts
// Using Model Updater for direct modification
const command = new PatchCommand('Create Node', modelUri, model => {
  const workflow = getValueByPointer(model, parentPath);
  const newNode = createNode(args.type, parentElement, args);
  workflow.nodes.push(newNode);
});
```

###### Command Stack IDs

Command Stack IDs define the scope of undo/redo operations. Typical scenarios include:

- One command stack per editor (for independent editing)
- Shared command stacks for interrelated models (for consistent state)

###### Model Hub Context

A Context defines the scope of a Model Hub instance, with completely independent contributions, model managers, and command stacks. Applications can define contexts based on:

- Project ID (for project isolation)
- Language ID (for language isolation)
- Or a single constant context for simpler applications

##### Validators

Register validators through a ValidationContribution:

```ts
@postConstruct()
protected init(): void {
  this.initialize({
    id: CUSTOM_SERVICE_KEY,
    validationContribution: new CustomValidationContribution(this.languageService)
  });
}
```

The ValidationContribution provides a list of Validators:

```ts
class CustomValidationContribution implements ModelValidationContribution {

  constructor(private languageService: CustomLanguageModelService){
    // Empty
  }

  getValidators(): Validator[] {
    return [
      new WorkflowValidator(),
      new TaskValidator()
    ];
  }
}

class WorkflowValidator implements Validator<string> {
  async validate(modelId: string, model: object): Promise<Diagnostic> {
    // Type guard to ensure this validator only handles relevant models
    if (isWorkflow(modelId, model)){
      // Validate workflow structure
    } else {
      return ok();
    }
  }
}

class TaskValidator implements Validator<string> {
  async validate(modelId: string, model: object): Promise<Diagnostic> {
    if (isWorkflow(modelId, model)){
      const tasks = model.nodes.filter(
        node => node.type === 'AutomaticTask' 
        || node.type === 'ManualTask');
      for (const task of tasks){
        // Validate each task
      }
    } else {
      return ok();
    }
  }
}
```

##### Custom APIs

Model Service Contributions expose public APIs for interaction with their models:

```ts
// Model Service definition
export interface CustomModelService {
  getCustomModel(modelUri: string): Promise<CustomModelRoot | undefined>;
  unload(modelUri: string): Promise<void>;
  edit(modelUri: string, patch: Operation[]): Promise<PatchResult>;
  createNode(modelUri: string, parent: string | Workflow, args: CreateNodeArgs): Promise<PatchResult>;
}

export const CUSTOM_SERVICE_KEY = 'customModelService';
```

The API can be accessed by any model hub client:

```ts
const customModelService = modelHub.getModelService<CustomModelService>(CUSTOM_SERVICE_KEY);
const modelUri = 'file:///custom-editor/examples/workspace/my.custom';
const customModel = await customModelService.getModel(modelUri);
const createArgs = {
  type: 'AutomaticTask',
  name: 'new task'
}
await customModelService.createNode(modelUri, customModel.workflows[0], createArgs);
```

### Form Implementation

<div style="text-align:center; margin-bottom:20px">
  <img src="../../images/treeeditor.png" alt="Overview of a Form with Model Hub" width="80%" />
</div>

Create sophisticated form based editors in Theia by leveraging [JSON Forms](https://jsonforms.io) and [ModelHub](https://github.com/eclipse-emfcloud/modelhub). This integration provides a robust foundation for creating customized form editors with seamless data management through the Frontend Model Hub.

#### Overview

The Theia Tree Editor framework provides extensive base classes and service definitions that can be extended and implemented to create tailored editors for specific data requirements. For comprehensive details, see the [official Theia Tree Editor documentation](https://github.com/eclipse-emfcloud/theia-tree-editor/blob/master/theia-tree-editor/DOCUMENTATION.MD).

#### Integrating ModelHub

Creating a Form Editor involves two key aspects:

1. **Data provisioning**: Use the `FrontendModelHub` to fetch and manage models
2. **Change synchronization**: Implement a `ModelService` to propagate editor changes back to the ModelHub

Achieve this integration by injecting the `FrontendModelHub` into your constructor and using it within the `init` method to monitor model changes and update the form accordingly. Implement the `handleFormUpdate` method to capture and apply data changes to the ModelHub-tracked model.

#### Example

Your widget should integrate the ModelHub by adding a listener within the `init` method:

```ts
this.modelHub
  .subscribe(this.getResourceUri()?.toString() ?? '')
  .then((subscription) => {
    subscription.onModelChanged = (_modelId, model, _delta) => {
      this.updateForm(model);
    };
  });
```

This listener ensures your form editor stays synchronized with any model changes in real-time.

Additionally, implement a `handleFormUpdate` method to propagate editor changes to the model:

```ts
protected override async handleFormUpdate(data: any): Promise<void> {
  const result = await this.modelService.edit(this.getResourceUri()?.toString() ?? '', data);
  this.updateForm(result.data);
}
```

This method ensures that any editor changes are immediately reflected in the model managed by the ModelHub.

The ModelService is a custom service with access to the ModelManager, see the [Custom APIs section]({{< relref "#custom-apis" >}}).

### Diagram Editor Implementation

<div style="text-align:center; margin-top:50px; margin-bottom:50px">
  <img src="../../images/diagramanimated.gif" alt="Overview of GLSP with Model Hub" width="80%" />
</div>

#### Introduction

Create powerful, interactive diagram editors using the [GLSP framework](https://eclipse.dev/glsp/) integrated with the [ModelHub](https://github.com/eclipse-emfcloud/modelhub). This combination provides a seamless foundation for sophisticated diagramming tools with robust data management. The GLSP diagram editor leverages the `ModelHub` and `ModelService` interfaces for optimal performance and maintainability.

#### ModelHub Integration

##### SourceModelStorage

The source model storage handles persistence of source models — loading from and saving to the model state. A source model is the underlying data model from which the diagram's graphical representation (graph model) is generated.

With ModelHub integration, these responsibilities are elegantly handled by the `ModelHub` itself, which provides:

- Model loading
- Subscription to model changes
- Dirty state tracking
- Model saving functionality

##### GModelFactory

The graph model factory transforms the source model (from the model state) into a graphical representation (`GModelRoot`). For complex transformations like creating edges, the `CustomModelService` provides reference resolution capabilities.

##### Model Operations

Operations on the model are forwarded to the [CustomModelService]({{< relref "#model-service-implementation" >}}), which provides specialized functions for model manipulation — such as dedicated create/delete functions that incorporate diagram-specific data like canvas positions.

#### Example

##### WorkflowModelStorage

The `WorkflowModelStorage` demonstrates how to load and save source models via the `ModelHub`:

###### Load the source model

Fetch the `CustomModelRoot` from the `modelHub`:

```typescript
this.modelHub.getModel<CustomModelRoot>(modelUri);
```

<details><summary>Detailed implementation</summary>

```typescript
  async loadSourceModel(action: RequestModelAction): Promise<void> {
    const modelUri = this.getUri(action);
    const customModel = await this.modelHub.getModel<CustomModelRoot>(modelUri);
    if (!customModel && customModel.workflows.length < 1)) {
      throw new GLSPServerError('Expected Model with at least one workflow');
    }
    this.modelState.setSemanticRoot(modelUri, customModel);
    this.subscribeToChanges(modelUri);
  }
```

</details>

###### Subscribe to model changes

Maintain real-time synchronization with model changes:

```typescript
this.subscription = this.modelHub.subscribe(modelUri);
this.subscription.onModelChanged = async (modelId: string, newModel: object) => {
  // handle model update
}
```

<details><summary>Detailed implementation</summary>

```typescript
private subscribeToChanges(modelUri: string): void {
    this.subscription = this.modelHub.subscribe(modelUri);
    this.subscription.onModelChanged = async (modelId: string, newModel: object) => {
      if (!newModel || newModel.workflows.length < 1) {
        throw new GLSPServerError('Expected Model with at least one workflow');
      }
      this.modelState.setSemanticRoot(modelId, newModel);
      const actions = await this.submissionHandler.submitModel();
      const dirtyStateAction = actions.find(action => action.kind === SetDirtyStateAction.KIND);
      if (dirtyStateAction && SetDirtyStateAction.is(dirtyStateAction)) {
        dirtyStateAction.isDirty = this.modelHub.isDirty(this.modelState.semanticUri);
      }
      this.actionDispatcher.dispatchAll(actions);
    };
  }
```

</details>

###### Save model

Trigger model saving through the ModelHub:

```typescript
this.modelHub.save(modelUri)
```

<details><summary>Detailed implementation</summary>

```typescript
async saveSourceModel(action: SaveModelAction): Promise<void> {
    const modelUri = action.fileUri ?? this.modelState.semanticUri;
    if (modelUri) {
      await this.modelHub.save(modelUri);
    }
  }
```

</details>

##### CustomGModelFactory

Resolve references using the `CustomModelServer` to properly translate the source model into a `GModelRoot`:

```typescript
  @inject(CustomModelServer)
  protected modelServer: CustomModelServer;

  ...

  protected async createEdge(edge: Flow): Promise<GEdge> {
    const [sourceNode, targetNode] = await Promise.all([
      this.modelServer.resolve<Node>(edge.source as Required<NodeReferenceInfo>),
      this.modelServer.resolve<Node>(edge.target as Required<NodeReferenceInfo>)
    ]);
    return GEdge.builder()
      .id(edge.id)
      .sourceId(sourceNode.id)
      .targetId(targetNode.id)
      .build();
  }
```

##### Model Operations via the CustomModelService

Forward model operations to the `CustomModelService` for clean, maintainable code:

```typescript
  override createCommand(operation: CreateNodeOperation): MaybePromise<Command | undefined> {
    return new CustomModelCommand(
      this.modelState,
      this.serializer,
      () => this.createWorkflowNode(operation)
    );
  }

  async createWorkflowNode(operation: CreateNodeOperation): Promise<void> {
    const container = this.modelState.semanticRoot;
    const modelService = this.modelHub.getModelService<CustomModelService>(CUSTOM_SERVICE_KEY);
    const modelId = this.modelState.semanticUri;
    await modelService?.createNode(modelId, container.workflows[0], {
      posX: this.getLocation(operation)?.x ?? Point.ORIGIN.x,
      posY: this.getLocation(operation)?.y ?? Point.ORIGIN.y,
      type: this.getNodeType(operation)
    });
  }
```

This approach works for all diagram editor operations including deleting elements, resizing, repositioning, and editing labels.

### Example Implementation

Explore detailed examples of the Model Hub approach in action:

https://github.com/eclipse-emfcloud/modelhub/tree/main/examples/theia

Be aware that the example does not cover all cases described above.
