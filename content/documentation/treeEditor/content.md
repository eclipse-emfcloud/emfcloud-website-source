+++
fragment = "content"
title = "Tree-based Editors"
weight = 180
[sidebar]
  sticky = true
+++

<div style="text-align:center; margin-bottom:20px">
  <img src="../../images/treeeditor.png" alt="Overview of the Tree Editor with Model Hub" width="80%" />
</div>

The foundational components for setting up tree-based editors in Theia are the [Theia Tree Editor](https://github.com/eclipse-emfcloud/theia-tree-editor) and [JSON Forms](https://jsonforms.io). This guide details how to effectively integrate these components along with the [ModelHub](https://github.com/eclipse-emfcloud/modelhub) to create a functional tree editor connected to the ModelHub. The Theia Tree Editor utilizes the Frontend Model Hub for seamless integration and data management.

## Overview

The Theia Tree Editor is a robust framework offering base classes and service definitions. These can be extended and implemented to tailor the editor for specific data requirements. For comprehensive information, refer to the [official Theia Tree Editor documentation](https://github.com/eclipse-emfcloud/theia-tree-editor/blob/master/theia-tree-editor/DOCUMENTATION.MD).

## Integrating ModelHub

Customizing the Theia Tree Editor involves providing data to the tree and setting up a listener for data updates within the editor. ModelHub primarily manages model provisioning. Utilize the `FrontendModelHub` from the ModelHub package to fetch models. To synchronize changes made in the editor back to ModelHub, leverage a `ModelService`. This service facilitates data updates to the model stored in ModelHub. Implement these functionalities by injecting `FrontendModelHub` into the constructor. Use it within the `init` method to monitor model changes and refresh the tree accordingly. Override the `handleFormUpdate` method to capture and apply data changes to the ModelHub-tracked model.

##  Example

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

The ModelService is a custom service which has access to the ModelManager, see the [Custom APIs section]({{< relref  "modelhub#custom-apis" >}}).
