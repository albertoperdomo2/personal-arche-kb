# Forge `extends` only applies to presets

In Forge, `extends` is interpreted by `ProjectConfig.apply_preset()` only while applying entries loaded from `presets.d/`. It is not a general inheritance keyword for arbitrary nested configuration such as `config.d/deployments.yaml`.

For llm-d deployment profiles, `get_deployment_profile()` already provides implicit inheritance by deep-merging `deployments.defaults` with the selected profile. Nested dictionaries merge recursively, but lists are replaced.

Practical consequence: represent composable settings such as `vllm_args` as dictionaries when profiles need to add a delta. A precise-prefix profile can then add only its specialized flags while inheriting common defaults. Adding an `extends` key to a deployment profile without implementing a dedicated resolver would merely leave an unused field in the resolved profile.