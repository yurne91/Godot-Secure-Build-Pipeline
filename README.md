# Godot 4.7.1 Secure Build Pipeline — .NET and GDScript

## Credits and upstream project

This build pipeline is based on **Godot Secure**, created and maintained by **KnifeXRage**:

https://github.com/KnifeXRage/Godot-Secure

A special thanks to KnifeXRage for creating and sharing Godot Secure with the Godot community. The custom PCK headers, security-token approach, Advanced Key Derivation, and the underlying Godot source modifications used by this pipeline build on that project.

If you use this workflow, please consider visiting the upstream repository, reading its documentation, and supporting or crediting the original author.

This repository/workflow is an integration and additional build-hardening layer around Godot Secure; it is **not the original Godot Secure project** and should not be presented as such.

This guide explains how to build a custom **Godot 4.7.1 editor and matching export templates** for either a **C#/.NET** or **GDScript** project using Godot Secure, AES-256 PCK encryption, encrypted PCK indexes, Advanced Key Derivation, and release hardening.

Two workflows are provided:

```text
build-godot-4.7.1-dotnet-secure-final.yml
build-godot-4.7.1-gdscript-secure-final.yml
```

Use the workflow that matches your project. Do not use the GDScript templates for a .NET project or the .NET templates for a non-.NET build.

> This is build hardening, not unbreakable DRM. Software running on an end user's machine can ultimately be inspected by a sufficiently skilled reverse engineer. The objective is to prevent the trivial standard-Godot extraction path and substantially increase the work required.

## 1. What the system does

Both variants start from a clean Godot 4.7.1 source tree, apply a pinned Godot Secure Universal v5 revision, build a custom editor, apply release hardening, and then build matching Windows and Linux release templates.

The secure build uses:

- Godot `4.7.1-stable`.
- Godot Secure Universal v5 at a pinned revision.
- AES-256 export encryption.
- Random custom PCK/encryption headers.
- A random 32-byte security token.
- Godot Secure Advanced Key Derivation.
- Split representation of the generated security token in the hardened source.
- Encrypted PCK index support.
- Release-only removal of selected encryption-related diagnostics and source-location breadcrumbs.
- Production compilation, LTO, disabled debug symbols, and binary stripping.

The **.NET workflow** additionally builds Godot with Mono/.NET support, generates the required glue and GodotSharp components, and packages the .NET editor/templates.

The **GDScript workflow** omits all Mono/.NET stages. It is therefore simpler and normally faster to build.

## 2. Requirements

You need:

- A private GitHub repository.
- GitHub Actions enabled.
- One of the supplied workflow YAML files, or both if you maintain both kinds of projects.
- A cryptographically random 256-bit key encoded as exactly 64 hexadecimal characters.

Never commit the key to Git.

## 3. Add the workflow

Create:

```text
.github/workflows/
```

For a .NET project, add:

```text
.github/workflows/build-godot-4.7.1-dotnet-secure-final.yml
```

For a GDScript project, add:

```text
.github/workflows/build-godot-4.7.1-gdscript-secure-final.yml
```

You do not need to commit the Godot source tree or Godot Secure repository. The workflows retrieve the pinned sources during the build.

## 4. Generate the AES-256 key

### Python

```python
import secrets
print(secrets.token_hex(32))
```

### OpenSSL

```bash
openssl rand -hex 32
```

The output must contain exactly 64 hexadecimal characters. Save it in a password manager or another secure secrets store.

## 5. Add the GitHub Actions secret

In your private repository open:

```text
Settings
→ Secrets and variables
→ Actions
→ New repository secret
```

Create:

```text
Name:   SCRIPT_AES256_ENCRYPTION_KEY
Secret: <your 64-character hexadecimal key>
```

The workflow validates the key format and masks it in Actions output.

## 6. Run the workflow

Open:

```text
Repository
→ Actions
→ select the secure build workflow
→ Run workflow
```

The workflow can also run on pushes to `main` if that trigger is left enabled.

Compilation can take a significant amount of time, particularly for the .NET variant.

## 7. Keep each editor/template set together

This rule is critical:

> **One workflow run = one matching editor/template set.**

Do not mix an editor from one secure build with templates from another build.

Godot Secure generates random build-specific material, including custom headers and a security token. Advanced Key Derivation can also generate build-specific derivation logic. Two builds using the same original AES key can therefore still be incompatible.

## 8. Artifacts — .NET

The .NET workflow produces artifacts equivalent to:

```text
godot-4.7.1-dotnet-windows-secure-editor
godot-4.7.1-dotnet-windows-encrypted
godot-4.7.1-dotnet-linux-encrypted
godot-4.7.1-secure-build-info
sha256-checksums
```

The Windows editor package includes the GodotSharp files required by the .NET editor.

Keep them together with the templates from the same Actions run.

## 9. Artifacts — GDScript

The GDScript workflow produces artifacts equivalent to:

```text
godot-4.7.1-gdscript-windows-secure-editor
godot-4.7.1-gdscript-windows-encrypted
godot-4.7.1-gdscript-linux-encrypted
godot-4.7.1-secure-build-info
sha256-checksums
```

There is no GodotSharp directory because Mono/.NET is not part of this build.

## 10. Use the custom editor

Extract the Windows secure editor somewhere permanent, for example:

```text
C:\Tools\GodotSecure-4.7.1\
```

Open your project with that custom editor when producing secure exports.

The editor and matching templates share the custom PCK format, security token, and key-derivation behavior.

## 11. Install the matching export templates

Extract the Windows/Linux template artifacts from the **same workflow run** and configure the custom editor to use those templates.

Do not use the official Godot export templates for a package produced by this secure pipeline.

## 12. Common encryption settings for .NET and GDScript

In the export preset use:

```text
Encryption
├─ Encrypt Exported PCK                 ON
├─ Encrypt Index (File Names and Info)  ON
├─ Filters to include                   *
└─ Filters to exclude                   [empty]
```

Enter the same original 64-character hexadecimal key stored in the GitHub secret.

Do not add `0x`, spaces, quotes, hyphens, or line breaks.

### Why `*`?

It makes all eligible exported package content match the encryption filter instead of protecting only selected extensions.

### Why encrypt the index?

Without index encryption, an extraction tool may still enumerate filenames and package structure even when individual files are encrypted.

## 13. Additional .NET export settings

For a production .NET export use:

```text
Dotnet
├─ Include Scripts Content    OFF
├─ Include Debug Symbols      OFF
└─ Embed Build Outputs        ON
```

`Include Scripts Content = OFF` avoids unnecessarily distributing the original C# source files.

`Include Debug Symbols = OFF` avoids shipping debug information in the production export.

`Embed Build Outputs = ON` places the .NET publication outputs into the exported package.

### Important .NET limitation

PCK encryption does not make a managed assembly permanently inaccessible. The .NET runtime ultimately has to load your game assembly. A determined user can potentially recover a materialized assembly or inspect it in memory and then use a .NET decompiler.

If protecting C# implementation details matters, treat **assembly obfuscation as a separate layer**. Test it carefully because Godot C# uses generated bindings and mechanisms that can be affected by aggressive renaming or trimming.

## 14. GDScript-specific notes

There are no Dotnet export settings to configure.

Your exported GDScript content, scenes, resources, and other eligible package data are protected by the same encrypted PCK pipeline when the include filter is `*`.

This avoids the separate managed-DLL exposure that exists in a C#/.NET game, although it still does not make runtime content impossible to recover.

## 15. Why the original export key may not work in a generic extractor

With this system, the original key is an input to the custom decryption scheme rather than the only piece of information involved.

Conceptually:

```text
Original 256-bit export key
             +
Generated security token
             +
Per-build Advanced Key Derivation
             ↓
Effective key used by the custom runtime
             ↓
Encrypted package
```

The matching custom editor and template contain compatible logic. A generic extraction tool that only receives the original key does not automatically know the generated token, custom derivation, or modified package headers.

This is why the game can run correctly while supplying only the original export key to a standard Godot extraction tool may still fail.

## 16. Clean runtime testing

Do not consider a successful Actions run sufficient. Export a real game build and test it thoroughly.

For both variants test:

1. Startup.
2. Main scene loading.
3. Several gameplay scenes.
4. `.res` and `.scn` resources.
5. Textures and shaders.
6. Audio.
7. Save/load functionality if applicable.
8. A complete gameplay session.
9. Closing and reopening the game.

For .NET additionally test C# initialization and, on Windows, perform at least one clean test after removing the corresponding runtime output directory under:

```text
%LOCALAPPDATA%\data_<project>_windows_x86_64\
```

This prevents files left by a previous run from hiding packaging problems.

## 17. Extraction-resistance test

A useful validation sequence is:

```text
Encrypted export
      ↓
Game runs correctly? ── No → investigate build/export configuration
      │
     Yes
      ↓
Try standard Godot extraction
      ↓
Package opens normally? ── Yes → verify encryption/index/filter/template
      │
      No
      ↓
Try the original export key
      ↓
Still not a standard extractable PCK? → expected with the custom secure format
```

Do not treat failure in one particular reverse-engineering utility as proof that the package is impossible to recover. It only demonstrates resistance to that extraction path.

## 18. Release hardening

The final workflows intentionally keep the editor useful for development while hardening the release templates.

The release stage removes selected encryption-specific error strings and source-location breadcrumbs, uses production-oriented compilation, LTO, disabled debug symbols, and stripping, and verifies that selected sensitive strings are not left in the resulting templates.

The pipeline deliberately avoids replacing Godot's cryptography with a large custom cryptographic implementation. Keeping modifications focused reduces runtime risk and makes future upgrades more manageable.

## 19. Security limitations

A sufficiently capable attacker controlling the client machine may still:

- reverse engineer the executable;
- inspect the package-loading path;
- debug or instrument the process;
- inspect decrypted memory;
- capture data after decryption;
- reconstruct the custom package format;
- patch the runtime;
- for .NET, recover and inspect managed assemblies.

The security goal is therefore to transform:

```text
standard Godot export
→ generic extractor
→ immediate project recovery
```

into something closer to:

```text
analyze custom executable
→ understand modified package format
→ understand build-specific derivation
→ inspect runtime behavior
→ adapt or write tooling
→ recover content
```

## 20. Updating Godot or Godot Secure

Both workflows intentionally pin versions. Do not casually update either dependency in a production pipeline.

For an upgrade:

1. Create a separate branch.
2. Change the Godot tag/revision.
3. Verify that the selected Godot Secure script explicitly supports that Godot version.
4. Run the workflow.
5. Do not bypass source-patch verification failures.
6. Build a fresh editor/template set.
7. Test real exports thoroughly.
8. Repeat extraction-resistance testing.
9. Only then promote the new toolchain to production.

The workflows are designed to fail when expected source patterns change. This is intentional; silently producing an unprotected or incompatible binary would be worse than a failed build.

## 21. Key rotation

To rotate the original AES key:

1. Generate a new 64-character hexadecimal key.
2. Replace the `SCRIPT_AES256_ENCRYPTION_KEY` repository secret.
3. Run the complete workflow again.
4. Download the new matching editor and templates.
5. Use the new original key in the export preset.
6. Never mix the old and new secure artifacts.

GitHub does not reveal a stored secret later, so keep your key in a proper secrets manager if reproducibility is required.

## 22. Repository hygiene

Recommended practices:

- Keep the repository private.
- Never commit the AES key.
- Never commit generated security tokens.
- Restrict access to Actions and release artifacts.
- Keep editor/templates from each workflow run together.
- Retain SHA-256 checksums for shipped toolchains.
- Record which workflow run produced each released game build.
- Avoid publishing diagnostic builds.
- Review changes before updating pinned dependencies.

## 23. Troubleshooting

### `Couldn't load project data`

Check first that the editor and template came from the same workflow run and that the export uses the correct original AES key.

Also verify that you did not accidentally use an official Godot template.

### `The MD5 sum of the decrypted file does not match`

The encrypted bytes did not decrypt to the expected original content. Common causes are a wrong key, mismatched secure editor/template artifacts, or mixing different secure builds.

The hardened production template removes some encryption-specific diagnostics, so use a diagnostic build when investigating difficult failures.

### The game opens and immediately closes

Run it from a terminal with verbose logging and inspect the first resource that fails to load. Do not keep changing multiple export settings at once; isolate one variable at a time.

### `.dll` / `Bad Image` errors in .NET

Clear the temporary Windows .NET runtime-output directory and export again. Verify the three recommended Dotnet settings and make sure the editor/template set is consistent.

### The game works but the original key does not work in a generic extractor

That can be expected. The matching secure runtime knows the generated token, custom headers, and Advanced Key Derivation used by that particular build; the original export key alone does not describe the entire modified scheme.

## 24. Production checklist — .NET

```text
Repository private                         YES
AES secret                                 exactly 64 hex characters
Godot version                              pinned
Godot Secure revision                      pinned
Editor + templates                         same workflow run

Dotnet / Include Scripts Content           OFF
Dotnet / Include Debug Symbols             OFF
Dotnet / Embed Build Outputs               ON

Encrypt Exported PCK                       ON
Encrypt Index                              ON
Filters to include                         *
Filters to exclude                         empty
Encryption Key                             same original AES key

Production template                        template_release
Debug symbols shipped                      NO
Checksums archived                         YES
```

## 25. Production checklist — GDScript

```text
Repository private                         YES
AES secret                                 exactly 64 hex characters
Godot version                              pinned
Godot Secure revision                      pinned
Editor + templates                         same workflow run

Encrypt Exported PCK                       ON
Encrypt Index                              ON
Filters to include                         *
Filters to exclude                         empty
Encryption Key                             same original AES key

Production template                        template_release
Debug symbols shipped                      NO
Checksums archived                         YES
```

## 26. Before shipping

Confirm all of the following:

- The game works from a clean environment.
- Representative scenes/resources/assets load correctly.
- The encrypted PCK option is enabled.
- The PCK index is encrypted.
- The encryption include filter is `*`.
- The custom editor and template belong to the same workflow run.
- No debug symbols are unintentionally distributed.
- A generic Godot extraction workflow cannot simply open the package.
- Supplying only the original export key does not turn the package back into a standard extractable PCK.
- For .NET, original `.cs` files are not included and managed-assembly exposure has been considered separately.
- The exact workflow YAML and SHA-256 checksums used for the release are archived.

---

## Disclaimer

This pipeline is a defense-in-depth measure for client-side Godot applications. It raises the cost of extraction and reverse engineering but cannot guarantee secrecy for code or data that must ultimately execute on an end user's machine.
