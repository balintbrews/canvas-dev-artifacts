# Canvas component-instance props form

A high-level overview of the Canvas editor's **props form**: the sidebar form
that appears when a content author selects a component instance and edits its
prop values.

> **Source snapshot:** Drupal Canvas `1.x` at commit [`6af7903d`][rev]
> (2026-11-11). This overview comes from reading the source at that revision;
> nothing was run against a live site. Code links are pinned to it, so line
> numbers will drift on newer commits.

## In short

- The form is a **Drupal Form API form rendered by React**. Drupal builds it,
  the `canvas_stark` theme renders it as custom elements, and the UI turns that
  markup into React components.
- For SDC and code components, each prop is edited through a **Drupal field
  widget**, chosen earlier by shape matching. Block components reuse the block
  plugin's own configuration form.
- Every input is wired to React Hook Form and Redux by **`withRHF`**. Per-widget
  **transforms** convert raw form values into prop values in the browser.
- A valid change is **PATCHed to the layout API**. The server stores it in the
  **auto-save** copy of the entity and returns fresh preview HTML.
- The real entity is only written when auto-saves are **published**, which is
  also where stored component inputs are fully validated.

## Architecture at a glance

```text
Canvas UI (React)                                  Drupal (PHP)
─────────────────                                  ────────────
ComponentInstanceForm.tsx ── PATCH form/component-instance/… ──▶ ComponentInstanceForm
                                                                   └─ component source builds widgets
                           ◀── form markup + transforms (JSON) ──
hyperscriptify → <DrupalForm> → inputs wrapped by withRHF
  └─ ComponentFormField: transforms → AJV check → debounce
       └─ PATCH layout/…  ──────────────────────────────────────▶ ApiLayoutController::patch()
                                                                   convert input → auto-save entity
                           ◀── preview HTML + layout + model ──
Publish (later): POST auto-saves/publish ───────────────────────▶ entity validation → save
```

## How it works

**1. Opening the form.** When a component instance is selected,
[`ComponentInstanceForm.tsx`][ui-cif] sends the instance's layout node and
current model to the [component-instance form route][r-cif] with a PATCH.
(Drupal AJAX inside the form POSTs to the same route.) Content templates and
patterns use [their own][r-cif-ct] [route variants][r-cif-pattern].

**2. Building it in Drupal.** [`ComponentInstanceForm`][php-cif] loads the
component at the version the instance uses and asks the component's source
plugin to build the form. SDC and code components get one field widget per prop,
plus each widget's transform metadata. Blocks get the block plugin's
configuration form. The form uses `#method = 'dialog'`, so it never submits
normally. Drupal returns the form as JSON: markup, assets and transforms.

**3. Rendering it in React.** The markup is converted into React components with
`hyperscriptify` and the [twig-to-JSX map][ui-map].
[`DrupalForm`][ui-drupalform] provides React Hook Form context, and Drupal
behaviors are re-attached so widgets such as autocomplete and the media library
keep working.

**4. Editing a value.** Each input is wrapped by [`withRHF`][ui-withrhf], which
hands it to [`ComponentFormField`][ui-cff]. On change, the value is recorded in
React Hook Form and Redux, transformed into a prop value, and checked against
the prop's JSON Schema where possible. Invalid values show an error and are not
saved. Valid changes are debounced (400 ms), and they also wait for any running
Drupal AJAX request.

**5. Saving, previewing and publishing.** The client PATCHes the instance's
model to the [layout API][r-lpatch]. The server converts it to stored input,
saves the entity's auto-save copy, and responds with preview HTML plus the
updated layout and model, which replace the client's copy. For code components,
simple prop changes can also be applied directly to the preview before the save
request is sent. Publishing auto-saves later validates and saves the real
entity.

## Boundaries

| Concern               | Where it lives                                                                                                                                                                                                                           |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Form structure**    | Server-built Form API form, fetched again when the selection, component version, undo/redo state or prop `source` changes.                                                                                                               |
| **Raw input state**   | React Hook Form, mirrored in the Redux `formState` slice. Cleared when the selection changes.                                                                                                                                            |
| **Instance model**    | Redux `layoutModel` slice: `{ source, resolved }` for SDC and code components, `{ resolved }` for blocks. Replaced by the server's response after each PATCH.                                                                            |
| **Preview**           | HTML returned by the PATCH. For code components, an optional in-place update of scalar props.                                                                                                                                            |
| **Persistence**       | Auto-save copy, written on each successful PATCH. The real entity is written only on publish.                                                                                                                                            |
| **Client validation** | AJV checks of SDC and code component props, with several exceptions ([details](#validation-details)). An invalid value blocks the PATCH.                                                                                                 |
| **Server validation** | Partly during editing: block form validation runs on form build and PATCH, and its errors are stored for later instead of failing the request. Full validation of stored inputs is enforced at publish ([details](#validation-details)). |

## Existing Canvas docs: what's stale

Checked at the same revision. The docs remain useful for intent and history;
these are the places where they no longer match the code.

- **[`redux-integrated-field-widgets.md`][doc-redux]**
  - Links to `shape-matching-into-field-types.md`, which no longer exists (now
    `shape-matching.md`). The same broken link is in `components.md`,
    `config-management.md` and `data-model.md`.
  - Points to `canvas.redux_integrated_field_widgets.inc`, which no longer
    exists. The hooks now live in
    `src/Hook/ReduxIntegratedFieldWidgetsHooks.php`.
  - The `withRHF.tsx` and `Form.tsx` paths are outdated.
  - The built-in transform list is missing `link` and `cast`.
- **[`component-and-entity-forms.md`][doc-forms]**
  - The `withRHF-fields/` paths are outdated (now `react-hook-form/fields/`).
  - It presents the layout PATCH as being for cases such as AJAX field addition.
    In the code, every valid edit goes through it.
  - It calls the Redux "blocking errors" server-side failures. In the code they
    are set only by client-side validation.
  - It says the preview POSTs on every model change. In the code, that only
    happens when the `updatePreview` flag is set, and the PATCH response clears
    that flag.

---

## Reference details (optional)

The rest of this page is for readers who want the internals.

### Routes and entry points

| Purpose                    | Default route                    | Content template                       | Pattern                            |
| -------------------------- | -------------------------------- | -------------------------------------- | ---------------------------------- |
| Build the form (PATCH)     | [form/component-instance][r-cif] | [variant][r-cif-ct]                    | [variant][r-cif-pattern]           |
| Save one instance (PATCH)  | [layout][r-lpatch]               | [layout-content-template][r-lpatch-ct] | [layout-pattern][r-lpatch-pattern] |
| Save the whole tree (POST) | [layout][r-lpost]                | `layout-content-template/…`            | `layout-pattern/…`                 |
| Publish                    | [auto-saves/publish][r-publish]  |                                        |                                    |

The UI mounts the form on `…/component/:componentId` in the entity, template and
pattern editors ([routes][ui-routes]). It requests the form via
[`componentInstanceForm.ts`][ui-formsvc] and saves via
[`updateComponent`][ui-updatecomponent].

### Building the form

- [`ComponentInstanceForm::buildForm()`][php-cif] resolves a host entity for
  input conversion ([code][php-cif-host]): the edited content entity, the
  content template's preview entity, or an empty stand-in for page variants.
  Patterns get none at this step.
- It converts the posted client model with `clientModelToInput()`, then calls
  the source's [`buildComponentInstanceForm()`][php-iface-build]. Broken
  components use the [`Fallback`][php-fallback] source, which shows the previous
  values read-only.
- **SDC and code components**
  ([`JsonSchemaPropsComponentSourceBase`][php-jsonbuild]):
  - Each prop's widget comes from the `Component` config entity's
    `prop_field_definitions` and is built by
    [`StaticPropSource::getWidget()`][php-getwidget].
  - Widgets that need an entity object get the host entity or a stand-in,
    including [`Pattern::createEmptyHostEntity()`][php-pattern-host].
  - Props using a non-static prop source are shown disabled, except
    `DefaultRelativeUrlPropSource`.
  - Content templates add prop-linker metadata and a [`linked_prop`
    element][php-linked] for linked props.
  - A widget without `canvas.transforms` metadata makes the build throw.
- **Blocks** ([`BlockComponent`][php-blockbuild]): the block plugin's
  configuration form, without `label`, `admin_label` and `label_display`.
- [`CanvasTemplateRenderer`][php-renderer] returns `html` (inside
  `<template data-hyperscriptify>`), `css`, `js`, `settings` and `transforms`.
  Form routes always use `canvas_stark` ([`applies()`][php-theme]).

### The client change pipeline

1. [`parseNewValue`][ui-parse] reads the value and applies transforms. The value
   is written to React Hook Form and to Redux `formState`.
2. [`validateNewValue`][ui-validate] runs AJV. An invalid value stores a field
   error and skips saving. So does an input carrying `data-canvas-no-update`.
3. If Drupal AJAX is running, the update waits for `drupalAjaxStop` and then
   goes through the same debounced store path ([code][ui-onchange]). The
   debounce is 400 ms ([`DEBOUNCE_TIMEOUT`][ui-debounce]); only checkbox and
   radio inputs of code-component scalar props skip it
   ([code][ui-shoulddebounce]).
4. [`createComponentFormStateHandler`][ui-handler] builds prop values with
   [`getPropsValues`][ui-getprops], adding a `cast` step for boolean and numeric
   props. It then syncs `source` to `resolved` and calls `patchComponent()`.
5. For string, number, integer and boolean props it first dispatches
   `canvas:component_preview_update`. The preview listener can only apply it to
   a `<canvas-island>` (code component) instance ([`IframeSwapper`][ui-island]).
   When it does, the PATCH is delayed by a further second
   ([`POLLED_BACKGROUND_TIMEOUT`][ui-polled]).
6. [`ApiLayoutController::patch()`][php-lpatch] checks the auto-save hashes,
   updates the instance with `clientModelToInput()` ([code][php-updateci]), and
   auto-saves. The client replaces its layout and model from the response with
   `updatePreview: false` ([code][ui-pessimistic]).

Inputs that Drupal AJAX responses render with `data-ajax` and `data-form-id`
trigger an `ajaxUpdateFormState` event ([dispatch][ui-ajaxdispatch]), which
[`ComponentInstanceForm.tsx`][ui-ajax] turns into the same kind of PATCH.

### Validation details

**Client side (AJV).** Runs in the change and blur handlers against
`propSources[prop].jsonSchema`. [`shouldSkipPropValidation`][ui-skip] skips:

- optional fields with an empty value, and optional selects set to `_none`
- inputs flagged with `data-canvas-no-validate`
- `object` props and arrays of objects
- props whose schema uses a Canvas `$ref` (for example colors)
- props that span multiple inputs

Props without a JSON schema, such as block settings, aren't checked. The React
Hook Form `Controller` rule validates a detached `<input>`, which this function
always skips, so in practice the handlers do the checking.

**Server side, during editing.**

- **SDC and code components:** form build and the layout PATCH convert input
  without a violations list, so `validateComponentInput()` is not called there.
- **Blocks:** `BlockComponent::clientModelToInput()` programmatically submits
  the block form ([code][php-block-submit]), which runs the plugin's
  `validateConfigurationForm()`. Errors (other than "required") are stored as
  component-instance form violations and then cleared, so the request still
  succeeds ([store][php-block-store]). A clean submit clears any stored
  violations ([code][php-block-clear]). Because `ComponentInstanceForm` also
  calls `clientModelToInput()`, this happens on form build as well as on PATCH.
- **Full-layout POST:** validation is not enforced
  ([`updateEntity()`][php-updateentity]). For content entities, the converter
  runs `$entity->validate()` but, with `validate: FALSE`, does not throw
  ([converter][php-converter], [validate call][php-converter-validate]).
  Config-entity hosts are converted without validation.

**Server side, at publish.** `POST /canvas/api/v0/auto-saves/publish` validates
config entities through typed data ([code][php-publish-config]) and content
entities through [`ensureEntityIsValid()`][php-ensurevalid]. The component
tree's `ValidComponentTreeItem` constraint calls each source's
[`validateComponentInput()`][php-validitem]. For SDC and code components that is
JSON Schema validation ([source][php-jsonvalidate]); for blocks it is config
schema validation plus the stored form violations ([source][php-blockvalidate]).

### Widgets and transforms

The widget for each SDC or code prop is stored in
`prop_field_definitions.<prop>.field_widget` when the component is discovered
(see Canvas's [`shape-matching.md`][doc-shape]). Transforms are declared in
`hook_field_widget_info_alter()` under `canvas.transforms`. The form adds
`multiple: true` when the prop's cardinality isn't 1. Client implementations
live in [`transforms.ts`][ui-transforms] and are exposed as
`Drupal.canvasTransforms` ([`main.tsx`][ui-main]). Modules can add their own
through `canvas.transform*` asset libraries ([aggregation][php-libalter],
[missing-library check][php-validatetransforms]).

Core mapping at this revision ([`fieldWidgetInfoAlter()`][php-widgetmap],
[`mediaLibraryFieldWidgetInfoAlter()`][php-mlmap]):

| Widget                                                           | Transforms                                                 |
| ---------------------------------------------------------------- | ---------------------------------------------------------- |
| `string_textfield`, `string_textarea`, `email_default`, `number` | `mainProperty`                                             |
| `boolean_checkbox`, `canvas_color_picker`                        | `mainProperty`                                             |
| `text_textfield`, `text_textarea`, `text_textarea_with_summary`  | `mainProperty` (`value`)                                   |
| `datetime_default`                                               | `mainProperty`, `dateTime`                                 |
| `daterange_default`                                              | `dateRange`                                                |
| `entity_reference_autocomplete`                                  | `mainProperty` (`target_id`), `entityAutocompleteTargetId` |
| `file_generic`, `image_image`                                    | `mainProperty` (`fids`)                                    |
| `link_default`                                                   | `link`                                                     |
| `media_library_widget`                                           | `mediaSelection`, `mainProperty` (`target_id`)             |
| `options_select`                                                 | none                                                       |

### Props definitions are a different UI

The code editor's "Props" panel ([`Props.tsx`]) edits prop _definitions_ for a
code component, not instance values. It is plain React state saved through the
`js_component` [config auto-save][r-cfgpatch]. It doesn't use Form API,
`withRHF` or transforms. It only affects the props form after the code component
is saved and its `Component` config entity gets new `prop_field_definitions`.

### Code map

| Area               | Start here                                                                                                                 |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------- |
| Form class         | [`src/Form/ComponentInstanceForm.php`][php-cif]                                                                            |
| Source contract    | [`ComponentSourceInterface`][php-iface-build]                                                                              |
| SDC and code props | [`JsonSchemaPropsComponentSourceBase`][php-jsonbuild], [`StaticPropSource`][php-getwidget]                                 |
| Blocks             | [`BlockComponent`][php-blockbuild]                                                                                         |
| Widget metadata    | [`ReduxIntegratedFieldWidgetsHooks.php`][php-widgetmap]                                                                    |
| UI entry           | [`ComponentInstanceForm.tsx`][ui-cif], [`componentInstanceForm.ts`][ui-formsvc]                                            |
| Field wiring       | [`withRHF.tsx`][ui-withrhf], [`ComponentFormField.tsx`][ui-cff], [`utils/index.ts`][ui-onchange]                           |
| Values, validation | [`componentFormData.ts`][ui-getprops], [`componentFieldValidation.ts`][ui-validate], [`transforms.ts`][ui-transforms]      |
| Saving, preview    | [`componentFormHandlers.ts`][ui-handler], [`preview.ts`][ui-updatecomponent], [`ApiLayoutController::patch()`][php-lpatch] |

<!-- Canvas source links, pinned to the examined revision. -->

[rev]:
  https://git.drupalcode.org/project/canvas/-/commit/6af7903d85e19e372155ccdc861719a45dfd6739
[doc-redux]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/docs/redux-integrated-field-widgets.md
[doc-forms]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/docs/component-and-entity-forms.md
[doc-shape]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/docs/shape-matching.md
[r-cif]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/canvas.routing.yml#L445
[r-cif-ct]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/canvas.routing.yml#L471
[r-cif-pattern]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/canvas.routing.yml#L488
[r-lpatch]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/canvas.routing.yml#L592
[r-lpatch-ct]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/canvas.routing.yml#L608
[r-lpatch-pattern]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/canvas.routing.yml#L622
[r-lpost]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/canvas.routing.yml#L631
[r-publish]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/canvas.routing.yml#L38
[r-cfgpatch]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/canvas.routing.yml#L121
[php-cif]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/Form/ComponentInstanceForm.php#L72
[php-cif-host]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/Form/ComponentInstanceForm.php#L89
[php-fallback]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/Plugin/Canvas/ComponentSource/Fallback.php#L120
[php-iface-build]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/ComponentSource/ComponentSourceInterface.php#L275
[php-jsonbuild]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/Plugin/Canvas/ComponentSource/JsonSchemaPropsComponentSourceBase.php#L904
[php-pattern-host]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/Plugin/Canvas/ComponentSource/JsonSchemaPropsComponentSourceBase.php#L929
[php-linked]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/Plugin/Canvas/ComponentSource/JsonSchemaPropsComponentSourceBase.php#L1042
[php-jsonvalidate]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/Plugin/Canvas/ComponentSource/JsonSchemaPropsComponentSourceBase.php#L726
[php-blockbuild]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/Plugin/Canvas/ComponentSource/BlockComponent.php#L398
[php-block-submit]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/Plugin/Canvas/ComponentSource/BlockComponent.php#L472
[php-block-store]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/Plugin/Canvas/ComponentSource/BlockComponent.php#L637
[php-block-clear]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/Plugin/Canvas/ComponentSource/BlockComponent.php#L649
[php-blockvalidate]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/Plugin/Canvas/ComponentSource/BlockComponent.php#L492
[php-getwidget]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/PropSource/StaticPropSource.php#L482
[php-renderer]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/Render/MainContent/CanvasTemplateRenderer.php#L97
[php-theme]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/Theme/CanvasThemeNegotiator.php#L46
[php-widgetmap]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/Hook/ReduxIntegratedFieldWidgetsHooks.php#L303
[php-mlmap]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/Hook/ReduxIntegratedFieldWidgetsHooks.php#L342
[php-libalter]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/Hook/ReduxIntegratedFieldWidgetsHooks.php#L46
[php-validatetransforms]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/Controller/CanvasController.php#L376
[php-lpatch]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/Controller/ApiLayoutController.php#L356
[php-updateci]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/Controller/ApiLayoutController.php#L780
[php-updateentity]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/Controller/ApiLayoutController.php#L818
[php-converter]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/ClientDataToEntityConverter.php#L71
[php-converter-validate]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/ClientDataToEntityConverter.php#L122
[php-publish-config]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/Controller/ApiAutoSaveController.php#L428
[php-ensurevalid]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/Controller/ApiAutoSaveController.php#L615
[php-validitem]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/src/Plugin/Validation/Constraint/ValidComponentTreeItemConstraintValidator.php#L94
[ui-routes]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/ui/src/app/AppRoutes.tsx#L134
[ui-cif]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/ui/src/components/ComponentInstanceForm.tsx#L285
[ui-ajax]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/ui/src/components/ComponentInstanceForm.tsx#L182
[ui-ajaxdispatch]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/ui/src/main.tsx#L309
[ui-formsvc]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/ui/src/services/componentInstanceForm.ts#L22
[ui-map]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/ui/src/components/form/twig-to-jsx-component-map.js#L39
[ui-drupalform]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/ui/src/components/form/components/drupal/DrupalForm.tsx#L23
[ui-withrhf]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/ui/src/components/form/react-hook-form/withRHF.tsx#L23
[ui-cff]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/ui/src/components/form/react-hook-form/fields/ComponentFormField.tsx#L47
[ui-shoulddebounce]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/ui/src/components/form/react-hook-form/fields/ComponentFormField.tsx#L92
[ui-onchange]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/ui/src/components/form/react-hook-form/utils/index.ts#L149
[ui-parse]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/ui/src/components/form/react-hook-form/fields/componentFieldValidation.ts#L267
[ui-validate]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/ui/src/components/form/react-hook-form/fields/componentFieldValidation.ts#L327
[ui-skip]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/ui/src/components/form/react-hook-form/fields/componentFieldValidation.ts#L118
[ui-debounce]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/ui/src/components/form/react-hook-form/fields/componentFormData.ts#L17
[ui-getprops]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/ui/src/components/form/react-hook-form/fields/componentFormData.ts#L226
[ui-handler]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/ui/src/components/form/react-hook-form/fields/componentFormHandlers.ts#L43
[ui-polled]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/ui/src/components/form/react-hook-form/fields/componentFormHandlers.ts#L14
[ui-transforms]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/ui/src/utils/transforms.ts#L456
[ui-main]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/ui/src/main.tsx#L162
[ui-updatecomponent]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/ui/src/services/preview.ts#L149
[ui-pessimistic]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/ui/src/services/preview.ts#L201
[ui-island]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/ui/src/features/layout/preview/IframeSwapper.tsx#L132
[`Props.tsx`]:
  https://git.drupalcode.org/project/canvas/-/blob/6af7903d85e19e372155ccdc861719a45dfd6739/ui/src/features/code-editor/component-data/Props.tsx
