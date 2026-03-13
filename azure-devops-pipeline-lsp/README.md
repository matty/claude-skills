# azure-devops-pipeline-lsp

Azure Pipelines language server for Claude Code, providing schema validation, completions, hover docs, and template navigation for Azure Pipelines YAML files.

## Supported Extensions
`.yml`, `.yaml`

## Installation

```bash
npm install -g azure-pipelines-language-server
```

## Schema

By default, the plugin uses the [public bundled schema](https://raw.githubusercontent.com/microsoft/azure-pipelines-vscode/main/service-schema.json) from the Azure Pipelines VS Code extension. This covers all standard pipeline constructs (stages, jobs, steps, triggers, resources, variables, built-in tasks, etc.).

### Org-specific schema

For validation that includes custom/marketplace tasks installed in your Azure DevOps organization, you can provide an org-specific schema. Fetch it from:

```
https://dev.azure.com/{ORG}/_apis/distributedtask/yamlschema
```

Save it locally and update the `settings.yaml.schemas` in your plugin config to point to the local file using a `file://` URI instead of the public URL.

## Requirements
- Node.js 16 or later

## File Detection

The language server receives all `.yml`/`.yaml` files, but the pipeline schema is only applied to files matching known Azure Pipelines conventions:

- `azure-pipelines.yml` / `.azure-pipelines.yml` (and `.yaml` variants)
- `vsts-ci.yml` / `.vsts-ci.yml` (legacy VSTS names)
- Files under `azure-pipelines/`, `.azure-pipelines/`, `.pipelines/`, or `pipelines/` directories

Non-matching YAML files (e.g. `docker-compose.yml`, Kubernetes manifests) get basic YAML parsing only — no false pipeline diagnostics.

If your pipeline files use a non-standard path, add the glob pattern to the `settings.yaml.schemas` array in the plugin config.

## Limitations
- Template references are navigational only (go-to-definition); template contents are not expanded or validated
- Compile-time expressions (`${{ }}`) are recognized syntactically but not evaluated
- Runtime expressions (`$[ ]`, `$()`) suppress type checking since their values are unknown at edit time

## More Information
- [azure-pipelines-language-server GitHub](https://github.com/microsoft/azure-pipelines-language-server)
- [Azure Pipelines YAML Reference](https://learn.microsoft.com/en-us/azure/devops/pipelines/yaml-schema)
