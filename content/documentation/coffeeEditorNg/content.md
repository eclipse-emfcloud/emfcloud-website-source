+++
fragment = "content"
title = "Modeling Tool Approaches"
weight = 130
[sidebar]
  sticky = true
+++

EMF Cloud provides two complementary approaches for implementing the model management. Each approach is tailored to specific requirements of modeling solutions and is integrated as a core part of the framework. On this page we provide an overview of both approaches and give recommendations which approach is the right for your modeling tool.

<div style="text-align:center; margin-bottom:20px">
  <img src="../../images/modelingtools.svg" alt="Overview of the Model Hub" width="70%" />
</div>

## Model Hub Approach

This approach focuses on a model-first, command-based model management. It is built around a central Model Hub that offers robust capabilities such as command-based editing, state management with undo/redo, and persistence. This is ideal for scenarios where fine-grained change control and model-oriented clients are most vital for your modeling tool. Detailed examples for this approach can be found in the Model Hub repository:

https://github.com/eclipse-emfcloud/modelhub/tree/main/examples/theia


## Langium-Based DSL Approach

This approach is designed for modeling tools that are built around textual domain-specific languages. Leveraging Langium, it provides advanced textual language features including parsing, reference resolution, workspace indexing, and language server integration, but extends it with a modeling-tool-oriented API to simplify accessing the models via an API for non-textual editors. This path is ideal for scenarios dealing with large interconnected models that are spread across many files and that require full control over the underlying textual format. For an example application with this architecture, please refer to the CrossModel tool, available at:

https://github.com/CrossBreezeNL/crossmodel

Most modeling tools can be built with either of the two approaches, however, many modeling tools have rather a text-file-oriented model management perspective or a model-oriented model management perspective, at least they tend in one or the other direction. Tending in the one or the other direction makes certain implementation strategies a more natural fit.

Instead of enforcing one approach for all modeling tools, EMF Cloud embraces these differences and provides a set of different components for both implementation strategies. Please use the table of contents on the left to navigate to the respective topics where you can find detailed guidance on implementation, customization, and best practices.

## Summary of Approaches

The table below highlights the key differences between the two approaches. For further guidance on integrating and customizing these approaches, please consult the dedicated sections in our documentation.


| Feature Category              | Model Hub                                     | Langium-Based DSL                           |
|-------------------------------|-----------------------------------------------|---------------------------------------------|
| Core Model Management         | ✓ Comprehensive management with command-based editing and undo/redo | ✓ Includes DSL-specific model management     |
| Model API                     | Advanced and flexible                         | Basic integration for DSL                    |
| Command-based Editing         | ✓ Built-in support for complex model edits    | -                                          |
| Multi-Client Support          | ✓ Real-time updates and coordination          | ✓ Supports multi-client scenarios            |
| Validation Framework          | Comprehensive model validation                | DSL-based text validation                    |
| Persistence Layer             | Customizable strategies                       | DSL-driven approaches                        |
| Language Infrastructure (LSP) | -                                             | ✓ Full Language Server Protocol support     |
| Reference Resolution          | Basic built-in resolution                     | Advanced symbol and reference management    |
