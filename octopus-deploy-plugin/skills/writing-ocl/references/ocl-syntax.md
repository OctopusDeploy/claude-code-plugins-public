# OCL syntax

OCL (Octopus Configuration Language) is a declarative configuration language modeled on HashiCorp Configuration Language (HCL). It is parsed by the `Octopus.Ocl` library; the round-trip serialization rules are enforced by the converters in `source/Octopus.Core/Serialization/Ocl/`.

## Block syntax

A block has a type, an optional label (or labels), and a body in braces:

```ocl
step "website-deployment" {
    name = "Website Deployment"
    // attributes and nested blocks here
}
```

- The type (`step`, `action`, `variable`, `value`, `connectivity_policy`, etc.) is a bare identifier.
- The label is a quoted string. For most blocks the label is the entity's slug (kebab-case), e.g. `step "deploy-kustomize"`.
- Some blocks take **two labels** (`packages "<reference-name>" { ... }`); some take **none** (`container { feed = ... image = ... }`); some optionally take one (`action` may be `action { ... }` or `action "<slug>" { ... }`).
- Empty blocks are legal: `connectivity_policy {}`.
- Blocks can nest arbitrarily deep.

## Attribute syntax

Attributes are `key = value` pairs:

```ocl
name = "My Step"
allow_deployments_to_no_targets = true
quantity_to_keep = 86
target_roles = ["WebServer", "AppServer"]
```

Keys are bare identifiers (or dotted identifiers like `Octopus.Action.X.Y` inside a `properties` map). Quote a key only if it contains characters that are not legal in identifiers.

## Value types

| Type | Form | Example |
|---|---|---|
| String (single-line) | Double-quoted | `name = "Deploy Web App"` |
| String (multi-line) | Heredoc | `script = <<-EOT\n  echo hi\n  EOT` |
| Boolean | bare keyword | `allow_deployments_to_no_targets = true` |
| Integer | bare digits | `quantity_to_keep = 86` |
| List | square brackets | `target_roles = ["WebServer", "AppServer"]` |
| Map | curly braces | `properties = { Octopus.Action.RunOnServer = "true" }` |
| Null / absent | omit attribute | (don't write `key = null`) |

In Octopus's OCL, **action property values are nearly always strings** even when they semantically represent booleans (`"True"` / `"False"`) or integers (`"30"`). The action property dictionary is typed as `Dictionary<string, string>` server-side, so `Octopus.Action.RunOnServer = true` (raw boolean) and `= "true"` (string) round-trip differently — the canonical form for properties is always **quoted string**. See examples in the canonical fixtures.

## Strings

Double-quoted strings support these escapes: `\\`, `\"`, `\n`, `\r`, `\t`, `\u{XXXX}`. Single quotes are literal characters; OCL does not have single-quoted strings.

### Octopus variable substitution

Octopus templates use `#{...}` syntax inside string values. **OCL leaves `#{...}` untouched** — it is expanded by Octopus at deployment time, not by the OCL parser.

```ocl
properties = {
    Octopus.Action.Azure.AccountId = "#{AutomationAccount}"
}
```

The slug-resolution layer also skips strings containing `#{`, so `worker_pool = "#{WorkerPool}"` is preserved verbatim and resolved at deployment time.

### HCL `${...}` interpolation

OCL technically supports `${...}` interpolation, but Octopus does not use it. Treat `${...}` as a literal in any string you produce.

## Heredocs (multi-line strings)

Use heredocs for any value that spans more than one line — script bodies, release notes, manual-intervention instructions, descriptions:

```ocl
properties = {
    Octopus.Action.Script.ScriptBody = <<-EOT
        Write-Host "Hello, world!"
        Invoke-RestMethod -Uri $url
        EOT
}
```

Rules:

- The opening token is `<<-EOT`. The `-` (dash-form heredoc) strips leading whitespace common to all lines, so the content can be indented for readability without affecting the parsed value.
- **Nothing after `<<-EOT` on the opening line** — the content starts on the next line.
- The closing `EOT` must be on its own line.
- The closing `EOT` should be indented to the same column as the content (the indentation it shares with content lines is what gets stripped).
- Use `EOT` as the delimiter unless you have a reason to choose another token. Other identifiers work (`<<-SCRIPT ... SCRIPT`), but `EOT` is the canonical Octopus convention.
- Plain (non-dash) heredocs `<<EOT ... EOT` do **not** strip indentation. Avoid them — they produce content with leading spaces.

Common gotcha: if the closing `EOT` is indented further than the content, OCL still strips only the common indentation. The result is content with negative indentation, which causes a parse error. Keep the closing `EOT` aligned with the content's left edge.

## Lists

```ocl
target_roles = ["WebServer", "AppServer"]
environments = ["production"]
excluded_environments = ["development", "staging"]
```

- Comma-separated, square brackets.
- Trailing commas are not required (and not canonical).
- An empty list is `[]`.
- Lists are heterogeneous in theory (mixed types), but every list in Octopus OCL is homogeneous strings.

## Maps

```ocl
properties = {
    Octopus.Action.Script.ScriptSource = "Inline"
    Octopus.Action.Script.Syntax = "PowerShell"
    Octopus.Action.RunOnServer = "True"
}
```

- Keys may be bare dotted identifiers (`Octopus.Action.RunOnServer`) or quoted strings.
- Values may be any value type, but in practice action `properties` are always strings.
- One entry per line is canonical. Avoid putting multiple entries on a single line.
- Order is not semantically meaningful, but the canonical form sorts entries alphabetically by key. The Octopus serializer emits them sorted, so prefer alphabetical order to minimize diff churn.

## Comments

```ocl
// line comment
# also a line comment
/*
 block comment
*/
```

Octopus's serializer does not emit comments, so any comments you add will be lost on the next round-trip through the server. Use comments only as authoring scratch — production OCL should be self-documenting.

## Whitespace

- Indentation is 4 spaces in canonical Octopus OCL fixtures.
- Blank lines are insignificant.
- The serializer always emits a final newline.

## Reserved tokens

Avoid using `true`, `false`, and bare numbers as block labels — quote them. The parser is generally tolerant, but the serializer will quote labels when round-tripping, so write quoted to begin with.

## Putting it together — a tiny canonical example

```ocl
step "say-hello" {
    name = "Say Hello"

    action {
        action_type = "Octopus.Script"
        environments = ["production"]
        properties = {
            Octopus.Action.RunOnServer = "True"
            Octopus.Action.Script.ScriptBody = <<-EOT
                Write-Host "Hello from #{Octopus.Environment.Name}!"
                EOT
            Octopus.Action.Script.ScriptSource = "Inline"
            Octopus.Action.Script.Syntax = "PowerShell"
        }
        worker_pool = "hosted-windows"
    }
}
```
