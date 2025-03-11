+++
fragment = "content"
title = "Model Hub Approach"
weight = 160
[sidebar]
  sticky = true
+++

The Model Hub Approach focuses on a model-first, command-based model management. It is built around a central Model Hub that offers robust capabilities such as command-based editing, state management with undo/redo, and persistence. This is ideal for scenarios where fine-grained change control and model-oriented clients are most vital for your modeling tool.

<div style="text-align:center; margin-bottom:20px">
  <img src="../../images/modelingtools.svg" alt="Overview of the Model Hub" width="70%" />
</div>

## Core Features

- **Command-based editing**: Support for complex model edits through a structured command system
- **Centralized state management**: Including undo/redo capabilities
- **Robust persistence layer**: Flexible storage strategies for models
- **Multi-client support**: Real-time updates and coordination across clients
- **Comprehensive model validation**: Built-in validation framework

## When to Choose This Approach

The Model Hub approach is particularly suited for:

- Modeling tools requiring fine-grained control over model changes
- Applications with complex editing operations that need undo/redo capabilities
- Solutions requiring real-time collaboration with multiple clients
- Projects where the model structure (rather than text files) is the primary focus

## Example Implementation

Detailed examples for this approach can be found in the Model Hub repository:

https://github.com/eclipse-emfcloud/modelhub/tree/main/examples/theia

## Tree Editor

<div style="text-align:center; margin-bottom:20px">
  <img src="../../images/treeeditor.png" alt="Overview of the Tree Editor with Model Hub" width="80%" />
</div>

The foundational components for setting up tree-based editors in Theia are the [Theia Tree Editor](https://github.com/eclipse-emfcloud/theia-tree-editor) and [JSON Forms](https://jsonforms.io). This guide details how to effectively integrate these components along with the [ModelHub](https://github.com/eclipse-emfcloud/modelhub) to create a functional tree editor connected to the ModelHub. The Theia Tree Editor utilizes the Frontend Model Hub for seamless integration and data management.

### Overview

The Theia Tree Editor is a robust framework offering base classes and service definitions. These can be extended and implemented to tailor the editor for specific data requirements. For comprehensive information, refer to the [official Theia Tree Editor documentation](https://github.com/eclipse-emfcloud/theia-tree-editor/blob/master/theia-tree-editor/DOCUMENTATION.MD).

### Integrating ModelHub

Customizing the Theia Tree Editor involves providing data to the tree and setting up a listener for data updates within the editor. ModelHub primarily manages model provisioning. Utilize the `FrontendModelHub` from the ModelHub package to fetch models. To synchronize changes made in the editor back to ModelHub, leverage a `ModelService`. This service facilitates data updates to the model stored in ModelHub. Implement these functionalities by injecting `FrontendModelHub` into the constructor. Use it within the `init` method to monitor model changes and refresh the tree accordingly. Override the `handleFormUpdate` method to capture and apply data changes to the ModelHub-tracked model.

###  Example

Your widget should extend the `NavigatableTreeEditorWidget` (similar to [ResourceTreeEditorWidget](https://github.com/eclipse-emfcloud/theia-tree-editor/blob/master/theia-tree-editor/src/browser/resource/resource-tree-editor-widget.ts)), and integrate ModelHub by adding a listener within the `init` method:

```ts
this.modelHub
  .subscribe(this.getResourceUri()?.toString() ?? '')
  .then((subscription) => {
    subscription.onModelChanged = (_modelId, model, _delta) => {
      this.updateTree(model);
    };
  });
```

This listener updates the tree editor in real-time with any changes in the data model.

Additionally, your widget should implement the `handleFormUpdate` method to trigger updates in the model using the ModelService:

```ts
protected override async handleFormUpdate(data: any, node: TreeEditor.Node): Promise<void> {
  const result = await this.modelService.edit(this.getResourceUri()?.toString() ?? '', data);
  this.updateTree(result.data);
}
```

This method ensures that any changes made in the editor are promptly reflected in the model managed by the ModelHub.

The ModelService is a custom service which has access to the ModelManager, see the [Custom APIs section]({{< relref  "#custom-apis" >}}).

## Diagram Editor

<div style="text-align:center; margin-top:50px; margin-bottom:50px">
  <img src="../../images/diagramanimated.gif" alt="Overview of GLSP with Model Hub" width="80%" />
</div>

### Introduction

The foundation of implementing diagram editors is the [GLSP framework](https://eclipse.dev/glsp/).
This guide will give an introduction to connect GLSP diagram editors with the [ModelHub](https://github.com/eclipse-emfcloud/modelhub).
Specifically, the GLSP diagram editor makes use of the `ModelHub` and the interfaces of the `ModelService` for seamless integration and outsourced data management.
Furthermore, a specific [example](#example) will be given as a guideline.

### ModelHub Integration

</br>

#### SourceModelStorage

A source model storage handles the persistence of source models, i.e. loading and saving it from/into the model state.
A `source model` is an arbitrary model from which the graph model of the diagram is to be created.
A source model loader obtains the information on which source model shall loaded from a
RequestModelAction; typically its client options. Once the source model is loaded, a model loader is expected
to put the loaded source model into the model state for further processing, such as transforming the loaded model
into a graph model (see [GModelFactory](#gmodelfactory) below).
On `saveSourceModel(SaveModelAction)`, the source model storage persists the source model from the model state.

In our case all of these responsibilites are outsourced to the `ModelHub`, namely the model loading, subscribing to model changes, dirty state information and triggering the save of the model.

#### GModelFactory

A graph model factory produces a graph model from the source model contained in the model state.

In this case, we also use the source model from the model state to translate the source model into `GModelRoot`. For more complex translations, e.g. for creating edges, the `CustomModelService` is used to resolve references.

#### Model Operations

To outsource the responsibility of executing operations directly on the model, operations should be forwarded to the [CustomModelService]({{< relref  "#model-service-implementation" >}}).
The `CustomModelService` offers tailored functions to manipulate the model, e.g. dedicated create or delete functions that take as arguments the specific diagram data like the position on the canvas.

### Example

</br>

#### WorkflowModelStorage

The `WorkflowModelStorage` is responsible for loading and saving source models via the `ModelHub`.

##### Load the source model

To load the source model for further usage and transformation into the graphical GModel, we fetch the `CustomModelRoot` from the `modelHub` as follows:

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

</br>

##### Subscribe to model changes

To subscribe to model changes, `modelHub` offers a `subscribe` function:

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

</br>

##### Save model

To trigger model saving, `modelHub` offers a `save` function:

```typescript
this.modelHub.save(modelUri)
```

<details><summary>Detailed implementation</summary

```typescript
async saveSourceModel(action: SaveModelAction): Promise<void> {
    const modelUri = action.fileUri ?? this.modelState.semanticUri;
    if (modelUri) {
      await this.modelHub.save(modelUri);
    }
  }
```
</details>

</br>

#### CustomGModelFactory

To resolve references, such as edge sources or targets, we use the `CustomModelServer` to get the resolved data needed to properly translate the source model into a `GModelRoot`.


This shows one example flow:

```json
  ...
  {
    "id": "checkWaterToDecision",
    "type": "Flow",
    "source": "Example.BrewingFlow.Check Water",
    "target": "Example.BrewingFlow.MyDecision"
  },
  ...
```

The following snippet shows how to create a GEdge from such a flow object:

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

#### Model Operations via the CustomModelService

To showcase how to forward model operations to the [CustomModelService]({{< relref  "#model-service-implementation" >}}),
let's take a look at the `CreateWorkflowNodeOperationHandler`.
This handler is an abstract node creation operation handler that is used to create all possible types of nodes supported by your diagram editor.

We wrap the model operation in a `GModelRecordingCommand` - in our case a customized `CustomModelCommand` that aligns the customized types of the `GModelState` for example.
The model operation itself is basically the operation data collection we get from the diagram editor, which we hand over to the `CustomModelService`.
The modelService provides a dedicated function to create a new node, expecting the diagram editor's specific data like position and type of node.

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

Similarly, this is the way to implement model operations, also for other diagram editor use cases like deleting model elements, resizing or repostioning model elements or edit model element labels.

## ModelHub

The EMF Cloud Model Hub is a central model management component that coordinates multiple clients, such as different editors, in their interaction and manipulation with models.
The Model Hub not only provides a generic API to access models, but is extensible with respect to different modeling languages.
Below we cover not only how clients can access models but also how new modeling languages can be registered.

### Interacting with models

An application may contain several model hubs, each associated to its own "context". The context is a unique string identifier. If an application requires a single model hub context, it may use a constant string; but it is also possible to use more dynamic values that take the current editor into account (e.g. using a folder path, or an application ID, or any value relevant to the application being developped).

Each instance of model hub comes with its own set of contributions, services, models and states.

To access the model hub for a given context, we use the ModelHubProvider, which is registered as a Theia Extension:

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

#### Loading and saving models

Loading and saving models can be achieved by calling the corresponding methods on your model hub instance, assuming contributions have been registered that can handled the requested model IDs. Model IDs are string identifiers that represent a model. They are typically URIs, but can be any arbitrary strings, as long as a Persistence Contribution is able to handle them (See [Persistence section]({{< relref  "#persistence" >}}) below).

```ts
const modelId = 'file:///custom-editor/examples/workspace/my.custom';
const model: object = await modelHub.getModel(modelId);
```

If you're certain about the type of your model, you can also directly cast it to the necessary type:

```ts
const modelId = 'file:///custom-editor/examples/workspace/my.custom';
const model: CustomModelRoot = await modelHub.getModel<CustomModelRoot>(modelId);
```

**Note:** If the model is already loaded, the in-memory instance will be immediately returned. Otherwise, the model hub will look for a Persistence Contribution that can handle the requested `modelId`, and load it before returning it. Since loading may require asynchronous operations, the `getModel()` method is itself asynchronous.

After applying some changes, you can save your model. For editing the model, the ModelHub uses Commands executed on a CommandStack, identified by a CommandStackId. When using a single model, the commandStackId can be the same value as the modelId. However, since Commands may affect multiple models in some cases, you may want to use a different CommandStackId. When saving this CommandStack, all models that have been modified by a Command executed on this CommandStack will be saved.

```ts
// In this example, we use a single model, so we can use the modelId as the commandStackId.
const modelId = 'file:///custom-editor/examples/workspace/my.custom';
const commandStackId = modelId;
modelHub.save(commandStackId);
```

Alternatively, you can save all modified models on all available Command Stacks:

```ts
modelHub.save();
```

#### Resolving references

If you have cross references in your model, i.e., pointers to other nodes either within the same model or another model in another document, you need to consider how those references should be represented and how they can be resolve to the referenced element.
By default, references are represented as information about a typed and named nodes located to a particular path within a document.
The main reasoning for this representation is that it uniquely identifies an element and can be serialized and sent to clients who can later use that information to resolve the actual element.
However, not all cross references may be resolvable due to changes in the model or the modeling language used.
In such cases, we want to provide as much information as we can to the client by at least giving the text that was used in the model for the reference and a potentially custom error.
Treating reference errors as just another case in reference resolutions allows them to be effectively handled by any type of client, no matter the visual representation.

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

While the ModelHub server flattens the reference information to be serializable, on the client side we often want to interact with the actual element that the reference represents instead of always being aware that there is a reference that we need to resolve.
To ease that more natural use of an object graph on the client side, we provide a utility function that replaces all unresolved references with reference objects that can query the object using a custom resolution mechanism.
In its purest form such a reference object may simply go to the ModelHub server and query the node based on the node info.
In more complex or high performance scenarios a different resolution or caching may be introduced.

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

#### Changing models

For the sake of isolation, the ModelHub doesn't expose methods to directly edit the models. Instead, Model Contributions are expected to register a Model Service, that will be responsible for handling all edition operations on the models it handles. Any application interesting in editing these models can request the corresponding Model Service, then call any of the exposed API methods to perform edit operations.

Edit Operations take the form of Commands, that are executed on a CommandStack. A Command may change one or several Models, and supports Undo/Redo operations.

Commands are usually handled directly by the Model Service, so they will not be visible to the client of the Model Service.

```ts
export interface CustomModelService {
  getCustomModel(modelUri: string): Promise<CustomModelRoot | undefined>;

  unload(modelUri: string): Promise<void>;

  edit(modelUri: string, patch: Operation[]): Promise<PatchResult>;

  createNode(modelUri: string, parent: string | Workflow, args: CreateNodeArgs): Promise<PatchResult>;
}

/**
 * This constant is used to register the CustomModelService and to retrieve
 * it from the ModelHub.
 */ 
export const CUSTOM_SERVICE_KEY = 'customModelService';

// Access and use the model service
const modelService: CustomModelService = modelHub.getModelService<CustomModelService>(CUSTOM_SERVICE_KEY);
await modelService.createNode(modelId, '/workflows/0', { type: 'AutomaticTask' });
```

#### Validating models

Model Contributions may register Validators. These Validators will be invoked whenever the ModelHub is validated:

```ts
const modelId = 'file:///custom-editor/examples/workspace/my.custom';
const diagnostic = await modelHub.validateModels(modelId);
```

Several models can be validated at the same time:

```ts
const modelId1 = 'file:///custom-editor/examples/workspace/foo.custom';
const modelId2 = 'file:///custom-editor/examples/workspace/my.custom';
const diagnostic = await modelHub.validateModels(modelId1, modelId2);
```

Or you can validate all models currently loaded, by omitting the `modelIds` argument:

```ts
const diagnostic = await modelHub.validateModels();
```

**Note:** In the latter case, only models currently *loaded* will be validated. Since the model hub relies on lazy-loading to identify existing models, it may ignore some models present in your workspace, if they have never been explicitly loaded beforehand.

The `validateModels` method will validate all requested models, then return the validation results, in the form of a Diagnostic.

If you're only interested in the latest known validation results, but don't want to wait for a full validation cycle, you can use `getValidationState` instead. This method doesn't trigger any validation, but returns the result from the latest validation:

```ts
const modelId = 'file:///custom-editor/examples/workspace/my.custom';
const currentDiagnostic = modelHub.getValidationState(modelId);
```

### Contributing modeling languages

The Model Hub can handle several aspects for each Modeling Language:

- Persistence (Save/Load)
- Edition (Via Model Services and Commands)
- Validation
- Triggers

All of these aspects can be registered using a `ModelServiceContribution`. The only mandatory aspect is Edition, via a Model Service.

```ts
/**
 * Our Model Service identifier. Used by clients to retrieve our Model Service.
 */
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

This minimal example lacks critical capabilities, that are required by most applications: persistence, and access to the Model Manager, in order to execute Commands. Here's a more complete and realistic example:

```ts
/**
 * Our Model Service identifier. Used by clients to retrieve our Model Service.
 */
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
    // Forward the model manager to our model service, so it can actually
    // execute some commands.
    this.modelService = new CustomModelServiceImpl(modelManager, this.languageService);
  }
}

class CustomPersistenceContribution implements ModelPersistenceContribution {
  
  constructor(private languageService: CustomLanguageModelService) {
    // Empty
  }

  canHandle(modelId: string): Promise<boolean> {
    // This example handles file URIs with the '.custom' extension
    return Promise.resolve(modelId.startsWith('file:/') 
      && modelId.endsWith('.custom'));
  }

  async loadModel(modelId: string): Promise<object> {
    // Load our model from file...
  }

  async saveModel(modelId: string, model: object): Promise<boolean> {
    // Save the new model to file...
  }
}
```

#### Persistence

Persistence is handled by specifying a `ModelPersistenceContribution` in your `ModelServiceContribution`.

```ts
  @postConstruct()
  protected init(): void {
    this.initialize({
      id: CUSTOM_SERVICE_KEY,
      persistenceContribution: new CustomPersistenceContribution(this.languageService)
    });
  }
```

The Persistence contribution needs to implement three methods: `canHandle(modelId)` to indicate which models it supports, `load(modelId)` and `save(modelId, model)` for the actual persistence.

```ts
class CustomPersistenceContribution implements ModelPersistenceContribution {

  constructor(private languageService: CustomLanguageModelService) {
    // Empty
  }
  
  canHandle(modelId: string): Promise<boolean> {
    // This example handles file URIs with the '.custom' extension
    return Promise.resolve(modelId.startsWith('file:/') 
      && modelId.endsWith('.custom'));
  }

  async loadModel(modelId: string): Promise<object> {
    // Load our model from file...
  }

  async saveModel(modelId: string, model: object): Promise<boolean> {
    // Save the new model to file...
  }
}
```

#### Cross References

As discussion in the [reference resolution section](#resolving-references), cross references that stem from your custom [Langium-based modeling language](../modelinglanguage/) need to be serializable so they can be sent to ModelHub clients.
Cross references in Langium are regular objects that may contain cycles.
Breaking those cycles is the main purpose of the [AstLanguageModelConverter](../langium/), a converter between the Langium-based AST model and the client language model.
By default, the ModelHub converter converts the `Reference` objects from Langium to `ReferenceInfo` objects that can be serialized and later revived again based on the document location:

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

As with all other services, the behavior of this conversion can be adapted by extending or completely replacing the implementation and re-binding it in the respective module.


#### Editing Domain

Edition of models is handled by the ModelManager, by executing Commands on one or several CommandStacks. The ModelManager is not directly exposed by the ModelHub API, so it is not (easily) possible to execute arbitrary commands on arbitrary models. Instead, the ModelManager is passed to ModelServiceContributions, which can then use it in their custom ModelService implementation to execute commands. This way, clients do not have to deal with Commands directly, but simply interact with the custom API.

##### Model Service implementation

As seen in the [ModelHub section]({{< relref  "#modelHub" >}}), the ModelServiceContribution is the entry point for model-specific contributions. Upon initialization of the ModelHub, each contribution will get access to the ModelManager, and should typically forward it to their ModelService implementation.

```ts
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
    return this.modelService as unknown as S;
  }

  setModelManager(modelManager: ModelManager): void {
    super.setModelManager(modelManager);
    // Forward the model manager to our model service, so it can actually
    // execute some commands.
    this.modelService = new CustomModelServiceImpl(modelManager);
  }
}
```

Then, the ModelService implementation can access the model and execute Commands on the ModelManager:

```ts
export class CustomModelServiceImpl implements CustomModelService {
  constructor(private modelManager: ModelManager<string>) {}

  // clients could directly access the model from the Model Hub, but 
  // custom ModelService APIs can also provide a convenience method:
  async getCustomModel(modelUri: string): Promise<CustomModelRoot | undefined> {
    const key = getModelKey(modelUri);
    return this.modelManager.getModel<CustomModelRoot>(key);
  }

  // [...]

  async createNode(modelUri: string, parent: string | Workflow, args: CreateNodeArgs): Promise<PatchResult> {
    const model = await this.getCustomModel(modelUri);
    if (model === undefined) {
      return {
        success: false,
        error: `Failed to edit ${modelUri.toString()}: Model not found`
      };
    }

    // parent can be either the element itself, or its path. Resolve the
    // correct element.
    const parentPath = this.getParentPath(model, parent);
    const parentElement = getValueByPointer(model, parentPath);
    if (!isWorkflow(parentElement)) {
      throw new Error(`Parent element is not a Workflow: ${parentPath}`);
    }

    // create the new Node (regular JSON object)
    const newNode = createNode(args.type, parentElement, args);

    // create a JSON patch to edit the model
    const patch: Operation[] = [
      {
        op: 'add',
        path: `${parentPath}/nodes/-`,
        value: newNode
      }
    ];
    
    // get the command stack. In this case, we don't have multi-model command stacks,
    // so just use the modelUri as the command stack id.
    const stackId = getStackId(modelUri);
    const stack = this.modelManager.getCommandStack(stackId);

    // create the Command from the JSON Patch and execute it
    const command = new PatchCommand('Create Node', modelUri, patch);
    const result = await stack.execute(command);

    // Return a patch result that indicates success or failure, and applied changes
    // in case of success.
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

There are several ways to create patch commands. In the above example, we created the JSON Patch manually, using the JSON pointer path from the parent, and adding a value. However, using a JSON Patch Library (such as `fast-json-patch`, although any similar library can be used), one could generate the patch instead:

```ts
export class CustomModelServiceImpl implements CustomModelService {

  // [...]

  async createNode(modelUri: string, parentPath: string): Promise<PatchResult> {
      const model = await this.getCustomModel(modelUri);

      // [...]

      // We are not allowed to edit the `model` object directly. Make a copy, 
      // and then we'll use fast-json-patch to generate a diff-patch.
      const updatedModel = deepClone(model) as CustomModelRoot;
      const updatedWorkflow = getValueByPointer(updatedModel, parentPath);
      if (!isWorkflow(parentElement)) {
        throw new Error(`Parent element is not a Workflow: ${parentPath}`);
      }

      // Directly modify the updatedModel object, by adding the new node to it
      const newNode = createNode(args.type, parentElement, args);
      updatedWorkflow.nodes.push(newNode);

      // Generate a patch using fast-json-patch.compare() and create a command
      const patch: Operation[] = compare(model, updatedModel);
      const command = new PatchCommand('Create Node', modelUri, patch);

      // [...]

      // then get the command stack and execute the command as before
      const result = await stack.execute(command);

      // [...]
  }
}
```

The ModelManager framework also provides a convenience method to create a PatchCommand that will directly edit the JSON Model, using a Model Updater:

```ts
export class CustomModelServiceImpl implements CustomModelService {

  // [...]

  async createNode(modelUri: string, parentPath: string): Promise<PatchResult> {
      const model = await this.getCustomModel(modelUri);

      // create a PatchCommand using a Model Updater. This allows us to directly
      // edit the model, without having to deal with JSON Patches at all.
      const command = new PatchCommand('Create Node', modelUri, model => {
        const workflow = getValueByPointer(model, parentPath);
        if (!isWorkflow(parentElement)) {
          throw new Error(`Parent element is not a Workflow: ${parentPath}`);
        }
        // Directly modify the updatedModel object, by adding the new node to it.
        // Inside of the Model Updater, we are allowed to edit the model object
        // directly - no need to create our own working copy!
        const newNode = createNode(args.type, parentElement, args);
        workflow.nodes.push(newNode);
      });

      // [...]

      // then get the command stack and execute the command as usual
      const result = await stack.execute(command);

      // [...]
  }
}
```

##### Command Stack IDs

A Command can be executed on any CommandStack. The Command Stack ID defines how Undo/Redo will behave, especially when a Command affects multiple models. When undoing (or redoing) changes on a Command Stack, the latest command executed on this Stack will be undone, ignoring commands that were executed in different stacks.

The most typical use case is to have one command stack per editor, so editors are independent from each other: undoing changes in one editor (command stack) does not affect the state of other editors.

However, when models have cross-references, you may be able to open inter-related models in different editors. In this case, it can be necessary to use a shared command stack for all editors, to ensure all editors work on a consistent state of the model. In that case, you may want to use a Command Stack Identifier that represents the entire set of inter-related models, such as the parent folder path, or project name.

##### Model Hub Context

A Context is a string identifier that defines the scope of a Model Hub. One model hub instance exists per context. Each Model Hub has its own set of Model Service Contributions, Model Manager, and set of Command Stacks, which are completely independent from other Model Hub instances. A Command executed in a given context cannot be undone from another context.

It is up to each application to decide how these contexts are defined. For example, you may decide that you need to isolate changes on a per-project basis, in which case it can be useful to use the Project ID as the Model Hub context. Alternatively, if you're defining several modeling languages without any relationship to each other, you may choose to use the language ID as the context.

For simpler applications, using a single, constant context ID is usually recommended.


#### Validators

A ModelServiceContribution can register a ValidationContribution, which will return a list of Validators. Validators will be invoked for all models (including the ones not actually handled by the Model Service Contribution), so they need to implement some kind of Type Guards to decide if they should actually try to validate a model or ignore it.

```ts
  @postConstruct()
  protected init(): void {
    this.initialize({
      id: CUSTOM_SERVICE_KEY,
      validationContribution: new CustomValidationContribution(this.languageService)
    });
  }
```

A ValidationContribution simply returns a list of Validators:

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
    // Start with a model typeguard, as all validators will be invoked
    // for all models.
    if (isWorkflow(modelId, model)){
      // Check that model is a well-formed Workflow...
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
        // Check that each Task is well-formed...
      }
    } else {
      return ok();
    }
  }
}
```

#### Custom APIs

Model Service Contributions may (and typically should) expose a public Model Service API, that can be used to interact with the models it provides. A Model Service is identified by a Key, and defined by an Interface. The implementation is then provided by the Model Service Contribution.

Model Service definition, exposed to all clients that may require it:

```ts
/**
 *  Our custom language-specific Model Service API
 */
export interface CustomModelService {
  getCustomModel(modelUri: string): Promise<CustomModelRoot | undefined>;

  unload(modelUri: string): Promise<void>;

  edit(modelUri: string, patch: Operation[]): Promise<PatchResult>;

  createNode(modelUri: string, parent: string | Workflow, args: CreateNodeArgs): Promise<PatchResult>;
}

/**
 * This constant is used to register the CustomModelService and to retrieve
 * it from the ModelHub.
 */ 
export const CUSTOM_SERVICE_KEY = 'customModelService';
```

Model Service contribution, used to register our language (minimal example):

```ts
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
    return this.modelService as unknown as S;
  }

  setModelManager(modelManager: ModelManager): void {
    super.setModelManager(modelManager);
    // Forward the model manager to our model service, so it can actually
    // execute some commands.
    this.modelService = new CustomModelServiceImpl(modelManager);
  }
}
```

The API can then be retrieved and used by any model hub client:

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
