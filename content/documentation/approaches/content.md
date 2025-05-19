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

This approach focuses on a model-first, command-based model management. It is built around a central Model Hub that offers robust capabilities such as command-based editing, state management with undo/redo, and persistence. This is ideal for scenarios where fine-grained change control and model-oriented clients are most vital for your modeling tool. 

For more details, see the [Model Hub Approach]({{< relref "/modelHubApproach" >}}) section.

## Langium-Based DSL Approach

This approach is designed for modeling tools that are built around textual domain-specific languages. Leveraging Langium, it provides advanced textual language features including parsing, reference resolution, workspace indexing, and language server integration, but extends it with a modeling-tool-oriented API to simplify accessing the models via an API for non-textual editors. 

For more details, see the [Langium-Based DSL Approach]({{< relref "/langiumApproach" >}}) section.

## Selecting the Right Approach

Most modeling tools can be built with either of the two approaches, however, many modeling tools have rather a text-file-oriented model management perspective or a model-oriented model management perspective, at least they tend in one or the other direction. Tending in the one or the other direction makes certain implementation strategies a more natural fit.

Instead of enforcing one approach for all modeling tools, EMF Cloud embraces these differences and provides a set of different components for both implementation strategies. 

## Summary of Approaches

The table below highlights the key differences between the two approaches. For further guidance on integrating and customizing these approaches, please consult the dedicated sections in our documentation.

<table>
  <thead>
    <tr>
      <th>Feature Category</th>
      <th>Model Hub</th>
      <th>Langium-Based DSL</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Core Model Management</td>
      <td>✓ Comprehensive management with command-based editing and undo/redo</td>
      <td>✓ Includes DSL-specific model management</td>
    </tr>
    <tr>
      <td>Model API</td>
      <td>Advanced and flexible</td>
      <td>Basic integration for DSL</td>
    </tr>
    <tr>
      <td>Command-based Editing</td>
      <td>✓ Built-in support for complex model edits</td>
      <td>-</td>
    </tr>
    <tr>
      <td>Multi-Client Support</td>
      <td>✓ Real-time updates and coordination</td>
      <td>✓ Supports multi-client scenarios</td>
    </tr>
    <tr>
      <td>Validation Framework</td>
      <td>Comprehensive model validation</td>
      <td>DSL-based text validation</td>
    </tr>
    <tr>
      <td>Persistence Layer</td>
      <td>Customizable strategies</td>
      <td>DSL-driven approaches</td>
    </tr>
    <tr>
      <td>Language Infrastructure (LSP)</td>
      <td>-</td>
      <td>✓ Full Language Server Protocol support</td>
    </tr>
    <tr>
      <td>Reference Resolution</td>
      <td>Basic built-in resolution</td>
      <td>Advanced symbol and reference management</td>
    </tr>
  </tbody>
</table>
