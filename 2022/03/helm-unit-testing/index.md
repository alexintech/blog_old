# Unit Testing for Helm Charts


Helm is a good tool to create common templates for your deployments. But when you start writing complex
templates testing them manually is unpleasure. Could I use TDD (Test-Driven Development) for 
writing Helm Charts?
<!--more-->

Basic testing includes such manual operations as:

1. Create `test-service-values.yml` with test values. Change that file for any case that you want to test.
2. Run something like this `helm template test-service spring-microservice --namespace esbs -f test-service-values.yml > test-service.yml`
3. Manually check the generated file `test-service.yml` that it's correct.

After some investigation I found three approaches to write unit tests for Helm Charts:

1. Helm unittest plugin;
2. Write test on bash using `bats` (example <https://github.com/hashicorp/vault-helm/tree/main/test/unit>);
3. HCUnit <https://github.com/xchapter7x/hcunit>

I chose the first approach as the most simple and readable. You can read test suite and assertion rules
on <https://github.com/quintush/helm-unittest/blob/master/DOCUMENT.md#>

Consider that we have created a basic Helm Chart. We want to add Vault Agent Injector capabilities
to the Chart.

## Using Helm unittest plugin

Start by installing `helm-unittest` plugin for Helm.
```shell
helm plugin install https://github.com/quintush/helm-unittest
```

For example our `deployment.yaml` from start was such:
```yaml
apiVersion: apps/v1
kind: Deployment
# ...
spec:
  # ...
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
# ...
```

### First iteration

First, create a directory called `tests` inside helm chart root folder. Then inside that folder create
a file called `vault_support_test.yaml`. Test files must include suffix `_test.yaml`. Add `tests` directory 
into `.helmignore` file.

```yaml
suite: test Vault Agent integration
templates:
  - deployment.yaml
tests:
  - it: should not generate Vault annotations if vault.enabled = false
    set:
      vault.enabled: false
    asserts:
      - isNull:
          path: spec.template.metadata.annotations
  - it: should set annotations from podAnnotations
    set:
      vault.enabled: false
      podAnnotations:
        test/some-annotation: "some-value"
    asserts:
      - equal:
          path: spec.template.metadata.annotations
          value:
            test/some-annotation: "some-value"
      - isNull:
          path: spec.template.metadata.annotations.[vault.hashicorp.com/agent-inject]
```

```ignore
# Patterns to ignore when building packages.
# This supports shell glob matching, relative path matching, and
# negation (prefixed with !). Only one pattern per line.
.DS_Store
# ...
tests
```

To run tests:
```shell
helm unittest --helm3 spring-microservice
```

```text
### Chart [ spring-microservice ] spring-microservice

 PASS  test Vault Agent integration     spring-microservice/tests/vault_support_test.yaml

Charts:      1 passed, 1 total
Test Suites: 1 passed, 1 total
Tests:       2 passed, 2 total
Snapshot:    0 passed, 0 total
Time:        59.915078ms
```

You can get an error if you're not using `--helm3` flag:
```text
### Error:  apiVersion 'v2' is not valid. The value must be "v1"


Charts:      1 failed, 1 errored, 0 passed, 1 total
Test Suites: 0 passed, 0 total
Tests:       0 passed, 0 total
Snapshot:    0 passed, 0 total
Time:        6.379641ms
```

Now let's add functionality to our Helm Chart step by step using TDD.

First, add new test to the suite:
```yaml
- it: should set annotations from podAnnotations and generated Vault annotations
  set:
    vault.enabled: true
    podAnnotations:
      test/some-annotation: "some-value"
  asserts:
    - equal:
        path: spec.template.metadata.annotations.[test/some-annotation]
        value: "some-value"
    - isNotNull:
        path: spec.template.metadata.annotations.[vault.hashicorp.com/agent-inject]
```

After running test we will get an error:
```text
 FAIL  test Vault Agent integration     spring-microservice/tests/vault_support_test.yaml
        - should set annotations from podAnnotations and generated Vault annotations

                - asserts[1] `isNotNull` fail
                        Template:       spring-microservice/templates/deployment.yaml
                        DocumentIndex:  0
                        Path:   spec.template.metadata.annotations.[vault.hashicorp.com/agent-inject]
                        Expected NOT to be null, got:
                                null


Charts:      1 failed, 0 passed, 1 total
Test Suites: 1 failed, 0 passed, 1 total
Tests:       1 failed, 2 passed, 3 total
```

Then we write simple code improvement to implement our new requirement:
```yaml
{{- if or .Values.podAnnotations .Values.vault.enabled }}
annotations:
  {{- if .Values.vault.enabled }}
  vault.hashicorp.com/agent-inject: true
  {{- end }}
  {{- toYaml .Values.podAnnotations | nindent 8 }}
{{- end }}
```

### Second iteration

Now write a new test:
```yaml
- it: should generate Vault annotations by default
  asserts:
    - equal:
        path: spec.template.metadata.annotations
        value:
          vault.hashicorp.com/agent-inject: true
          vault.hashicorp.com/agent-image: "registry-test.alfa-bank.kz/esbs/docker-base-images/vault:1.9.2-curl"
          vault.hashicorp.com/preserve-secret-case: true
          vault.hashicorp.com/ca-cert: "/vault/tls/ca.crt"
          vault.hashicorp.com/tls-secret: "vault-tls-client"
          vault.hashicorp.com/role: "k8s-test-default-role"
```

For that test we will add new values to `values.yaml`. That values used by default:
```yaml
vault:
  enabled: true
  image: "registry-test.alfa-bank.kz/esbs/docker-base-images/vault:1.9.2-curl"
  role: "k8s-test-default-role"
  debug: false
```

Running test we will get an error:
```text
 FAIL  test Vault Agent integration     spring-microservice/tests/vault_support_test.yaml
        - should generate Vault annotations by default
                Error: yaml: line 26: did not find expected key


Charts:      1 failed, 0 passed, 1 total
Test Suites: 1 failed, 0 passed, 1 total
Tests:       1 failed, 1 errored, 3 passed, 4 total
```

You can see that it's not just failed (means that assertions are failed), but *errored*. That tells us
to look at code on line 26. In my example, if `.Values.podAnnotations` is empty, then this error arises.
Let's fix it:
```yaml
{{- if or .Values.podAnnotations .Values.vault.enabled }}
annotations:
  {{- if .Values.vault.enabled }}
  vault.hashicorp.com/agent-inject: true
  vault.hashicorp.com/agent-image: "registry-test.alfa-bank.kz/esbs/docker-base-images/vault:1.9.2-curl"
  vault.hashicorp.com/preserve-secret-case: true
  vault.hashicorp.com/ca-cert: "/vault/tls/ca.crt"
  vault.hashicorp.com/tls-secret: "vault-tls-client"
  vault.hashicorp.com/role: "k8s-test-default-role"
  {{- end }}
  {{- with .Values.podAnnotations }}
    {{- toYaml . | nindent 8 }}
  {{- end }}
{{- end }}
```

#### Refactoring

Now we can see 4 passed test, without any failed or errored. We can refactor this code to simplify 
maintainability. Create `templates/_vault.tpl` file with the following content:
```yaml
{{/*
Sets pod annotations with vault injector annotations
*/}}
{{- define "spring-boot-microservice.podAnnotations" -}}
      {{- if or .Values.podAnnotations .Values.vault.enabled }}
      annotations:
        {{- if .Values.vault.enabled }}
          {{- include "spring-boot-microservice.vaultAnnotations" . | nindent 8 }}
        {{- end }}
        {{- with .Values.podAnnotations }}
          {{- toYaml . | nindent 8 }}
        {{- end }}
      {{- end }}
{{- end }}

{{/*
 Sets vault injector annotations
*/}}
{{- define "spring-boot-microservice.vaultAnnotations" -}}
vault.hashicorp.com/agent-inject: true
vault.hashicorp.com/agent-image: "registry-test.alfa-bank.kz/esbs/docker-base-images/vault:1.9.2-curl"
vault.hashicorp.com/preserve-secret-case: true
vault.hashicorp.com/ca-cert: "/vault/tls/ca.crt"
vault.hashicorp.com/tls-secret: "vault-tls-client"
vault.hashicorp.com/role: {{ .Values.vault.role | quote }}
{{- end }}
```

And change `templates/deployment.yaml` to simple one:
```yaml
apiVersion: apps/v1
kind: Deployment
# ...
spec:
  # ...
  template:
    metadata:
      {{- template "spring-boot-microservice.podAnnotations" . }}
# ...
```

### Third iteration

Next we need to generate such annotations with Go template as values. Add them to the last test:
```yaml
vault.hashicorp.com/agent-inject-secret-config.properties: "k8s-prod/data/test/test-service"
vault.hashicorp.com/agent-inject-template-config.properties: |
  {{- with secret "/k8s-prod/data/test/test-service" -}}
    {{- range $k, $v := .Data.data -}}
      {{- if not ($k | regexMatch "base64file") -}}
        {{- $k }}={{ printf "%s\n" $v -}}
      {{- end -}}
    {{- end -}}
  {{- end -}}
```

{{< admonition tip "Escaping Go Template in Helm Chart" >}}
To escape Go template in the Helm Chart, we will use the solution from <https://github.com/helm/helm/issues/2798> 
{{< /admonition >}}

Insert inside `templates/_vault.tpl`:
```yaml
vault.hashicorp.com/agent-inject-secret-config.properties: "k8s-prod/data/test/test-service"
vault.hashicorp.com/agent-inject-template-config.properties: |
  {{`{{- with secret "/k8s-prod/data/test/test-service" -}}
    {{- range $k, $v := .Data.data -}}
      {{- if not ($k | regexMatch "base64file") -}}
        {{- $k }}={{ printf "%s\n" $v -}}
      {{- end -}}
    {{- end -}}
  {{- end -}}`}}
```

### Forth iteration

Rewrite `.Release.Name` for our last test, and set `fullnameOverride` value:
```yaml
- it: should generate Vault annotations by default
  release:
    namespace: test
  set:
    fullnameOverride: "test-service"
  asserts:
#...
```

Change `templates/_vault.tpl` like this:
```yaml
{{- define "spring-boot-microservice.vaultAnnotations" -}}
#...
vault.hashicorp.com/agent-inject-secret-config.properties: {{ include "spring-boot-microservice.vaultSecretPath" . }}
vault.hashicorp.com/agent-inject-template-config.properties: |
  {{`{{- with secret "/k8s-prod/data/test/test-service" -}}
    {{- range $k, $v := .Data.data -}}
      {{- if not ($k | regexMatch "base64file") -}}
        {{- $k }}={{ printf "%s\n" $v -}}
      {{- end -}}
    {{- end -}}
  {{- end -}}`}}

{{- end }}


{{/* Get Vault secret path */}}
{{- define "spring-boot-microservice.vaultSecretPath" -}}
{{- printf "%s/data/%s/%s" "k8s-prod"
          .Release.Namespace
          (include "spring-microservice.fullname" . ) }}
{{- end }}
```

### Fifth iteration

Now we want to define Vault KV secret based on `environment` variable. If it set to `prod`, `k8s-prod` KV
should be used.

```yaml
- it: should use Vault KV based on environment setting
  release:
    namespace: test-ns
  set:
    fullnameOverride: "test-service"
    environment: prod
  asserts:
    - equal:
        path: spec.template.metadata.annotations.[vault.hashicorp.com/agent-inject-secret-config.properties]
        value: "k8s-prod/data/test-ns/test-service"
```

Add the following configuration settings into `values.yaml`:
```yaml
environment: prod

vaultKV:
  test: "k8s-test"
  prod: "k8s-prod"
```

Update `templates/_vault.tpl` in a such way:
```yaml
{{- define "spring-boot-microservice.vaultSecretPath" -}}
{{- printf "%s/data/%s/%s"
          (include "spring-boot-microservice.vaultKV" . )
          .Release.Namespace
          (include "spring-microservice.fullname" . ) }}
{{- end }}

{{/*
Define Vault KV (key-value) engine based on selected environment (test|prod)
*/}}
{{- define "spring-boot-microservice.vaultKV" -}}
{{- if .Values.environment }}
{{- default "" (printf "%s" (index .Values.vaultKV .Values.environment)) }}
{{- end }}
{{- end }}
```

### Templates issues

Suppose that we want to use predefined templates for Vault templating. And that templates will be not
Strings, but another template that we evaluate using `tpl` function. For example, our `values.yaml`
will be changed:
```yaml
vault:
  enabled: true
  files:
    config.properties:
      secretPath: '{{ include "spring-boot-microservice.vaultSecretPath" . }}'
      secretKey: ""
      useTemplate: "kv"
  templates:
    kv: |
      {{`{{- with secret`}} {{ .secretPath | quote }} {{`-}}
        {{- range $k, $v := .Data.data -}}
          {{- if not ($k | regexMatch "base64file") -}}
            {{- $k }}={{ printf "%s\n" $v -}}
          {{- end -}}
        {{- end -}}
      {{- end -}}`}}
    file: |
      {{`{{- with secret` }} {{ .secretPath | quote }} {{ `-}}
        {{- base64Decode (index .Data.data `}}{{ .secretKey | quote }}{{`) }}
      {{- end -}}`}}
```

As you can see, `vault.templates.kv` contains `.secretPath` variable, and `vault.templates.file` 
additionally evaluates `.secretKey` variable. The values of that variables should be gotten from 
`vault.files.*.secretPath` and `vault.files.*.secretKey` respectively.

Let's write a test:
```yaml
- it: should set Vault annotations to load a binary file
  release:
    namespace: test-ns
  set:
    fullnameOverride: "test-service"
    vault.files:
      GOST_cert.p12:
        secretPath: "/k8s-prod/data/test-ns/test-service"
        secretKey: "service.eds.base64file"
        useTemplate: "file"
  asserts:
    - equal:
        path: spec.template.metadata.annotations.[vault.hashicorp.com/agent-inject-secret-GOST_cert.p12]
        value: "/k8s-prod/data/test-ns/test-service"
    - equal:
        path: spec.template.metadata.annotations.[vault.hashicorp.com/agent-inject-template-GOST_cert.p12]
        value: |
          {{- with secret "/k8s-prod/data/test-ns/test-service" -}}
            {{- base64Decode (index .Data.data "service.eds.base64file") }}
          {{- end -}}
    - equal:
        path: spec.template.metadata.annotations.[vault.hashicorp.com/agent-inject-secret-config.properties]
        value: "/k8s-prod/data/test-ns/test-service"
```

Trying to implement such logic, we can try this snippet:
```yaml
{{- range $fileName, $secretParams := .Values.vault.files }}
{{ tpl (index $.Values.vault.templates (index $secretParams "useTemplate"))
        (dict "secretPath" (tpl $secretParams.secretPath $)
        "secretKey" $secretParams.secretKey
        ) | indent 2 }}
{{- end }}
```

We can send to the `tpl` function dictionary (map) of key-values, that will be used in the final 
template. But we will get errors:
```text
- should set Vault annotations to load a binary file
                Error: template: spring-microservice/templates/_vault.tpl:8:14: executing "spring-boot-microservice.podAnnotations" at <include "spring-boot-microservice.vaultAnnotations" .>: error calling include: template: spring-microservice/templates/_vault.tpl:34:3: executing "spring-boot-microservice.vaultAnnotations" at <tpl (index $.Values.vault.templates (index $secretParams "useTemplate")) (dict "secretPath" (tpl $secretParams.secretPath $) "secretKey" $secretParams.secretKey)>: error calling tpl: cannot retrieve Template.Basepath from values inside tpl function: {{`{{- with secret` }} {{ .secretPath | quote }} {{ `-}}
  {{- base64Decode (index .Data.data `}}{{ .secretKey | quote }}{{`) }}
{{- end -}}`}}
: "BasePath" is not a value
```

The solution has been found on <https://github.com/helm/helm/issues/5979#issuecomment-704740061>. 
We need to send extra argument `"Template" $.Template` to `tpl` function. Fixed code looks like this:
```yaml
{{- range $fileName, $secretParams := .Values.vault.files }}
{{- $useTemplate := index $secretParams "useTemplate" }}
{{- $secretPath := (tpl $secretParams.secretPath $) }}
vault.hashicorp.com/agent-inject-secret-{{ $fileName }}: {{ $secretPath }}
vault.hashicorp.com/agent-inject-template-{{ $fileName }}: |
{{ tpl (index $.Values.vault.templates $useTemplate)
        (dict "secretPath" $secretPath
        "secretKey" $secretParams.secretKey
        "Template" $.Template) | indent 2 }}
{{- end }}
```

### Using multiple templates in unit tests

Previously in our test suite we have specified only one `deployment.yaml` in the list `templates`. That
means only one template will be generated for tests.

Consider a case when we want to generate `ServiceAccount` even if `serviceAccount.create` disabled
but `vault.enabled` is true. If we specify new test like this:
```yaml
- it: should enable serviceAccount when vault is enabled
  set:
    fullnameOverride: "test-service"
    serviceAccount.create: false
    vault.enabled: true
  asserts:
    - containsDocument:
        kind: ServiceAccount
        apiVersion: v1
        name: test-service
      template: serviceaccount.yaml
```

It will fail with an error:
```text
- asserts[1] `containsDocument` fail
    Error:
            template "spring-microservice/templates/serviceaccount.yaml" not exists or not selected in test suite
```

This error tells us that when you specify `template` value for specific assert or test, it must be 
included in the list of `templates` in a test suite. So to not overwrite previous tests, that used 
only `deployment.yaml` we will create separate test suite `tests/serviceaccount_test.yaml`. 
It contains all variations of configuration settings that we want:
```yaml
suite: test ServiceAccount
templates:
  - deployment.yaml
  - serviceaccount.yaml
tests:
  - it: should enable serviceAccount when vault is enabled
    set:
      fullnameOverride: "test-service"
      serviceAccount.create: false
      vault.enabled: true
    asserts:
      - equal:
          path: spec.template.spec.serviceAccountName
          value: "test-service"
        template: deployment.yaml
      - containsDocument:
          kind: ServiceAccount
          apiVersion: v1
          name: test-service
        template: serviceaccount.yaml

  - it: should create serviceAccount when serviceAccount.create enabled
    set:
      fullnameOverride: "another-service"
      serviceAccount.create: true
      vault.enabled: false
    asserts:
      - equal:
          path: spec.template.spec.serviceAccountName
          value: "another-service"
        template: deployment.yaml
      - containsDocument:
          kind: ServiceAccount
          apiVersion: v1
          name: another-service
        template: serviceaccount.yaml

  - it: should not create serviceAccount when serviceAccount.create disabled
    set:
      serviceAccount.create: false
      vault.enabled: false
    asserts:
      - equal:
          path: spec.template.spec.serviceAccountName
          value: "default"
        template: deployment.yaml
      - hasDocuments:
          count: 0
        template: serviceaccount.yaml
```

Another trick to implement such behaviour is setting value of `serviceAccount.create` to `true`
if `vault.enabled` is `true` using `set` function:
```yaml
{{- if .Values.vault.enabled -}}
{{- $_ := set .Values.serviceAccount "create" true }}
{{- end -}}
{{- if .Values.serviceAccount.create -}}
apiVersion: v1
kind: ServiceAccount
#...
{{- end }}
```

