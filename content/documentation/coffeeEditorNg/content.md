+++
fragment = "content"
title = "Modeling Tool Approaches"
weight = 130
[sidebar]
  sticky = true
+++

EMF Cloud provides two complementary approaches for building modeling tools. Each approach is tailored to specific requirements and is integrated as a core part of the framework.

## Model Hub Approach

This approach focuses on comprehensive model management. It is built around a central Model Hub that offers robust capabilities such as command-based editing, state management with undo/redo, and persistence. This is ideal for scenarios where a powerful environment for managing complex models is required. Detailed examples for this approach can be found in the Model Hub repository:

https://github.com/eclipse-emfcloud/modelhub/tree/main/examples/theia

<div style="text-align:center; margin-bottom:20px">
  <img src="../../images/coffeeeditormodelhub.svg" alt="Overview of the Model Hub" width="70%" />
</div>

## Langium-Based DSL Approach

This approach is designed for applications that require dedicated DSL support. Leveraging Langium, it provides advanced textual language features including parsing, reference resolution, and language server integration. This path is ideal for scenarios where the development of a domain-specific language is crucial. A prominent example demonstrating this approach is the CrossModel application, available at:

https://github.com/CrossBreezeNL/crossmodel

Both approaches are integral parts of EMF Cloud, offering distinct solutions tailored to different modeling requirements. Please use the table of contents on the left to navigate to the respective topics where you can find detailed guidance on implementation, customization, and best practices.

## Summary of Approaches

This summary table encapsulates the key differences between the two approaches. The Model Hub approach is ideal for scenarios demanding robust, command-based model management, while the Langium-Based DSL approach excels in projects that require advanced textual language support and full Language Server Protocol (LSP) integration. For further guidance on integrating and customizing these approaches, please consult the dedicated sections in our documentation.

Below is a summary table highlighting the key differences between the two approaches:

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
